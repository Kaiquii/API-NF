# Arquitetura modular do MVP

> Status: planejamento concluído, aguardando implementação.

## Decisão

O MVP da API-NF será construído como um **monólito modular em Go**, executado em processos separados para API HTTP e worker assíncrono. Os processos compartilharão os mesmos módulos de domínio, contratos internos e banco PostgreSQL, mas terão permissões e responsabilidades próprias.

Microsserviços, broker externo obrigatório e orquestração distribuída não fazem parte do MVP. Eles poderão ser avaliados quando houver evidência de volume, isolamento ou organização de equipe que justifique o custo adicional.

## Motivo da escolha

A emissão fiscal exige consistência entre autenticação, idempotência, numeração, estado do documento, XML, comunicação com a SEFAZ e armazenamento de artefatos. No primeiro MVP, manter esses componentes no mesmo produto reduz a complexidade operacional sem misturar responsabilidades no código.

Essa escolha preserva evolução futura porque os módulos já possuem fronteiras explícitas. Um componente só será extraído para serviço independente quando existir um motivo mensurável, como carga isolada, equipe dedicada, requisito de disponibilidade ou limite técnico comprovado.

## Visão geral

```text
ERP ou sistema do cliente
          |
          v
API HTTP
  - autenticação e autorização
  - validação de entrada
  - idempotência
  - criação do documento e da outbox
          |
          v
PostgreSQL + armazenamento privado
          |
          v
Worker assíncrono
  - normalização e regras fiscais
  - numeração e estados
  - XML e assinatura A1
  - SEFAZ e reconciliação
  - webhooks e artefatos fiscais
```

O worker recebe somente identificadores confiáveis. Certificados, senhas, payloads completos e XMLs não devem viajar livremente em filas ou logs.

## Processos executáveis

```text
/cmd/api
  Servidor HTTP externo. Recebe chamadas do ERP, autentica a API key,
  valida a solicitação, aplica idempotência, persiste a transação e responde.

/cmd/worker
  Consome eventos da outbox, processa o documento fiscal, comunica-se com a
  SEFAZ, grava resultado e entrega webhooks.

/cmd/admin
  Executa migrations, provisiona tenants, administra configurações e realiza
  operações internas controladas. Não é exposto como API pública.
```

A API HTTP não deve aguardar a autorização da SEFAZ. Uma solicitação aceita retorna um recurso consultável; o worker continua o processamento fora da conexão do cliente.

## Organização sugerida do código

```text
/cmd
  /api
  /worker
  /admin

/internal
  /platform
  /auth
  /tenancy
  /companies
  /catalog
  /fiscal
  /nfe
  /ingestion
  /issuance
  /sefaz
  /artifacts
  /webhooks
  /operations
```

`internal` é a fronteira principal do produto. Não haverá pacote público genérico antes de existir necessidade real de compartilhamento com outro produto.

## Módulos e responsabilidades

### `platform`

Infraestrutura compartilhada: configuração por ambiente, conexão PostgreSQL, migrations, armazenamento privado, criptografia, relógio, geração de IDs, logs, métricas, traces e encerramento controlado dos processos.

Não contém regra fiscal ou regra de negócio específica.

### `auth`

Valida API keys, aplica expiração, revogação, rotação e limites de uso. Produz a identidade autenticada que será usada pelos demais módulos, sem expor o segredo da chave.

### `tenancy`

Resolve o grupo autenticado, seu schema e as regras de isolamento. Todo acesso a empresa, integração, documento ou artefato deve usar esse contexto; um identificador isolado nunca autoriza acesso entre tenants.

### `companies`

Administra empresas emitentes, dados cadastrais oficiais, séries, ambientes, certificado A1 e referências seguras para segredos. O ERP seleciona a empresa pelo `company_id`, mas não substitui seus dados fiscais oficiais no envio da nota.

### `catalog`

Mantém dados de apoio, como produtos e destinatários, quando o cenário permitir enriquecimento controlado. Cadastros podem completar dados conhecidos, mas não podem alterar silenciosamente fatos comerciais recebidos do ERP.

### `fiscal`

Contém o modelo fiscal canônico, normalização, validação estrutural, regras comuns, snapshots, manifestos de cenário e contratos internos. É independente de HTTP, banco, XML e SEFAZ.

### `nfe`

Implementa o módulo fiscal `nfe/v1`: regras e estruturas específicas da NF-e modelo 55, composição fiscal, geração dos dados que alimentam o XML e critérios de cobertura do cenário habilitado.

Novos documentos fiscais, como NFC-e, NFS-e, CT-e e MDF-e, serão módulos próprios; não devem ser adicionados como condicionais dispersas dentro de `nfe`.

### `ingestion`

Expõe o contrato público da API, valida o JSON recebido, aplica mapping declarativo quando houver integração customizada e produz o DTO público normalizado. O mapping pode transformar formato e tipos, mas não executa código arbitrário nem modifica regras fiscais.

### `issuance`

É o núcleo do fluxo de emissão. Controla idempotência, referência externa, máquina de estados, reserva de numeração, tentativas, outbox, leases e retentativas. Documento, estado e evento da outbox são gravados na mesma transação.

### `sefaz`

Gera e valida XML, aplica assinatura A1, seleciona o serviço fiscal habilitado, transmite, consulta recibos, interpreta retornos e reconcilia resultados desconhecidos. Recebe um snapshot fiscal pronto; não reinterpreta cadastro ou regras atuais de um documento já enfileirado.

### `artifacts`

Protege XMLs, protocolos, respostas relevantes, hashes e metadados. Os arquivos ficam em armazenamento privado; o PostgreSQL preserva referências, integridade e controle de acesso.

### `webhooks`

Entrega eventos finais para o cliente usando a outbox transacional. Controla assinatura, versão de evento, tentativas, backoff, bloqueio de destinos inseguros e auditoria de entrega. A consulta do documento permanece como fonte de verdade.

### `operations`

Reúne auditoria, métricas, alertas, dashboards, runbooks e operações administrativas mínimas. Todos os componentes devem propagar IDs de correlação para permitir rastrear uma emissão da API até a SEFAZ e o webhook.

## Regras de dependência

1. Cada módulo possui contratos próprios e não acessa diretamente detalhes internos de outro módulo.
2. O domínio fiscal não depende de HTTP, PostgreSQL, SDK de armazenamento ou cliente da SEFAZ.
3. Adaptadores externos dependem do domínio e implementam interfaces definidas pelo caso de uso, nunca o contrário.
4. `nfe` depende de contratos de `fiscal`; `fiscal` não depende de `nfe`.
5. `sefaz` recebe o snapshot fiscal e devolve resultado técnico; não escolhe impostos, altera cadastro nem cria regras tributárias.
6. `webhooks` reage a eventos persistidos; não altera o resultado fiscal de uma emissão.
7. A API e o worker usam permissões de banco diferentes e não executam DDL em sua inicialização.
8. Toda operação multi-tenant recebe contexto de grupo antes de consultar ou alterar dados.

## Fluxo de emissão

```text
POST da integração
  -> auth valida API key
  -> tenancy resolve grupo e schema
  -> ingestion valida e normaliza payload
  -> fiscal monta snapshot do cenário
  -> issuance aplica idempotência e grava documento + outbox
  -> API responde 202 Accepted

worker consome outbox
  -> issuance assume lease e reserva numeração quando aplicável
  -> nfe compõe o documento fiscal
  -> sefaz gera, valida, assina e transmite XML
  -> artifacts preserva XML e protocolo
  -> issuance atualiza o estado
  -> webhooks entrega evento final
```

Falhas após uma chamada externa não autorizam retransmissão cega. O worker deve consultar e reconciliar o resultado antes de criar nova tentativa.

## Dados e infraestrutura do MVP

- Go para API, worker e comando administrativo;
- PostgreSQL 18 com `pgx/v5` e `pgxpool`;
- migrations SQL-first com Goose e comando próprio;
- schema `platform` e schemas isolados por grupo;
- outbox e leases no PostgreSQL; broker externo é opcional no futuro;
- armazenamento de objetos privado para XMLs e demais artefatos;
- criptografia envelope ou serviço equivalente para certificados e segredos;
- logs estruturados, métricas e traces desde o Marco 1.

## Capacidades do MVP

- NF-e modelo 55 em recorte fiscal explicitamente habilitado;
- contrato JSON padrão e entrada customizada por mapping;
- API keys por grupo e ambiente;
- grupos, empresas e integrações isolados por tenant;
- validação sem emissão;
- idempotência, numeração e processamento assíncrono;
- XML, assinatura A1, autorização e reconciliação SEFAZ;
- consulta de documento e download autorizado de XML;
- cancelamento e inutilização para o cenário habilitado;
- webhooks finais assinados;
- logs, métricas, alertas, auditoria e suporte operacional básico;
- piloto produtivo controlado.

## Fora do escopo inicial

- microsserviços independentes;
- broker externo obrigatório;
- Kubernetes e operação multirregião;
- NFC-e, NFS-e, CT-e, MDF-e e cobertura fiscal nacional;
- painel administrativo visual completo;
- DANFE em PDF, carta de correção e contingência produtiva automática;
- múltiplos endpoints de webhook por integração;
- analytics avançado e suporte 24x7 sem escala formalizada.

## Critérios para reconsiderar microsserviços

Uma extração só será considerada quando houver evidência objetiva, por exemplo:

- carga de workers que prejudique a API mesmo após ajuste de concorrência;
- necessidade comprovada de escalonamento ou deploy independente;
- domínio estabilizado com contratos internos maduros;
- equipe responsável por uma capacidade específica;
- exigência de isolamento técnico ou regulatório que o monólito modular não atenda.

Sem esses sinais, módulos bem delimitados no mesmo repositório e processos separados permanecem a opção preferencial.

## Relação com os marcos de implementação

- M1 cria os três comandos, configuração, qualidade e observabilidade básica.
- M2 implementa banco, schemas, migrations e isolamento.
- M3 implementa autenticação, contexto do tenant e proteção de certificados.
- M4 a M9 implementam o fluxo fiscal, emissão, XML, SEFAZ e artefatos.
- M10 completa webhooks, operação e suporte.
- M11 valida prontidão e executa o piloto controlado.

## Decisão final

O projeto seguirá com monólito modular em Go, API e worker separados, PostgreSQL como base transacional e outbox como mecanismo assíncrono do MVP. A primeira implementação deve priorizar o caminho completo de uma NF-e do cenário piloto, antes de ampliar módulos fiscais, cobertura tributária ou distribuição de serviços.
