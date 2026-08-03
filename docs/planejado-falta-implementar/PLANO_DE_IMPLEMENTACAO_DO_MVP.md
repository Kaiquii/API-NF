# Plano de implementacao do MVP

## Objetivo

Transformar as decisoes arquiteturais, fiscais, operacionais e de seguranca das Etapas 1 a 10 em uma sequencia executavel de entregas para o primeiro MVP da plataforma fiscal.

Este plano define:

- recorte do produto inicial;
- dependencias e caminho critico;
- marcos tecnicos;
- backlog de alto nivel com identificadores estaveis;
- criterios de entrada e saida;
- testes e evidencias obrigatorios;
- portoes de homologacao e producao;
- riscos e decisoes externas bloqueadoras;
- capacidades conscientemente adiadas.

O objetivo nao e estimar datas antes de reduzir os principais riscos. O progresso sera medido por capacidades demonstradas e criterios de aceite atendidos.

## Estado deste planejamento

As decisoes tecnicas necessarias para ordenar a implementacao estao consolidadas. O projeto pode iniciar pelo Marco 0 sem escolher antecipadamente uma empresa, UF ou regime ficticio.

UF, regime tributario, operacao fiscal, empresa piloto e certificado A1 dependem de informacoes reais. Esses dados formam um portao externo obrigatorio do Marco 0. Sua ausencia nao reabre a arquitetura, mas bloqueia a implementacao das regras especificas e a homologacao real com a SEFAZ.

## Principios de execucao

- Entregar fatias verticais verificaveis, nao apenas camadas desconectadas.
- Reduzir cedo os riscos de XML, assinatura, certificado e comunicacao com a SEFAZ.
- Manter o recorte fiscal inicial estreito e explicitamente habilitado.
- Construir isolamento multi-tenant mesmo quando o piloto usar um unico grupo.
- Tratar seguranca, auditoria, testes e observabilidade como parte de cada entrega.
- Nunca usar producao para descobrir se um fluxo fiscal funciona.
- Mocks e fixtures aceleram desenvolvimento, mas nao substituem homologacao real.
- Uma falha de comunicacao nunca autoriza retransmissao cega.
- Nenhum marco pode depender de alteracao manual direta no banco.
- Adiar capacidade nao essencial sem criar decisao estrutural dificil de reverter.
- Priorizar o caminho critico e limitar trabalho simultaneo, especialmente enquanto o projeto tiver equipe pequena.

## Recorte do primeiro MVP

### Incluido

- NF-e modelo 55;
- entrada JSON pelo contrato padrao;
- entrada customizada por mapeamento declarativo;
- autenticacao maquina a maquina por API key;
- grupos, empresas e integracoes isolados por tenant;
- uma UF, um regime e um conjunto inicial de cenarios explicitamente homologados;
- certificado digital A1;
- ambiente de homologacao e producao separados;
- normalizacao, enriquecimento, validacao e snapshot fiscal;
- numeracao atomica e idempotencia obrigatoria;
- processamento assincrono por outbox e worker;
- geracao, validacao e assinatura de XML;
- autorizacao, consulta e reconciliacao com a SEFAZ;
- cancelamento e inutilizacao para o recorte habilitado;
- armazenamento privado do XML e protocolo;
- consulta publica do documento e download autorizado do XML;
- webhooks finais assinados;
- logs, metricas, traces, alertas, runbooks e auditoria;
- operacoes administrativas minimas e protegidas;
- piloto produtivo controlado.

### Nao incluido na primeira ativacao

- NFC-e modelo 65;
- NFS-e, CT-e, MDF-e ou outros modulos fiscais;
- cobertura nacional ou de todos os regimes;
- painel administrativo visual completo;
- DANFE em PDF;
- carta de correcao, salvo nova decisao explicita;
- contingencia produtiva automatica;
- multiplos endpoints de webhook por integracao;
- regras de negocio arbitrarias no mapping;
- broker externo obrigatorio;
- microsservicos independentes;
- operacao ativa em multiplas regioes;
- analytics fiscal ou comercial;
- suporte 24x7 sem plantao formalizado.

## Manifesto obrigatorio do piloto

Antes de implementar regras fiscais especificas ou executar homologacao real, o arquivo ou registro de manifesto do piloto devera conter:

| Campo | Estado no inicio deste plano | Criterio |
| --- | --- | --- |
| Modulo fiscal | Definido | `nfe` |
| Versao do contrato | Definido | `nfe/v1` |
| Modelo | Definido | `55` |
| Ambiente inicial | Definido | Homologacao |
| Modalidade inicial | Definido | Emissao normal |
| UF do emitente | Pendente de dado real | Uma UF suportada e registrada. |
| Autorizador aplicavel | Derivado da UF | Catalogo e versao registrados. |
| Regime tributario | Pendente de dado real | Regime confirmado por responsavel fiscal. |
| Tipo de operacao | Pendente de dado real | Operacao interna ou interestadual claramente definida. |
| Finalidade e consumidor | Pendente de dado real | Cenario comercial e destinatario definidos. |
| Tributos e beneficios | Pendente de dado real | ICMS, PIS, COFINS, IPI e beneficios aplicaveis revisados. |
| Empresa de homologacao | Pendente de dado real | CNPJ, IE, endereco e habilitacao validos. |
| Certificado A1 | Pendente de dado real | PFX valido, senha disponivel por canal seguro e cadeia confiavel. |
| Produtos de teste | Pendente de dado real | NCM, unidade, origem e tributacao revisados. |
| Responsavel fiscal | Pendente de nomeacao | Pessoa que aprova o manifesto e as evidencias. |
| Responsavel tecnico | Pendente de nomeacao | Pessoa que executa e registra a homologacao. |

O manifesto sera versionado. Alterar UF, regime, operacao, beneficio ou regra que influencie o XML cria nova versao e exige avaliacao de cobertura e novos testes.

Uma empresa real valida o cenario, mas nao define a estrutura generica do produto.

## Estrategia de entrega

### Caminho critico

```text
M0 Recorte e prova tecnica
  -> M1 Fundacao de engenharia
  -> M2 Banco e multi-tenancy
  -> M3 Autenticacao e configuracao segura
  -> M4 Nucleo fiscal offline
  -> M5 Ingestao e mapeamento
  -> M6 Estado, numeracao e processamento assincrono
  -> M7 XML e assinatura
  -> M8 Integracao e homologacao SEFAZ
  -> M9 Eventos e artefatos fiscais
  -> M10 Operacao, webhooks e suporte
  -> M11 Prontidao e piloto controlado
```

Observabilidade, seguranca, testes e documentacao atravessam todos os marcos. M10 completa essas capacidades, mas nao e o primeiro momento em que elas aparecem.

### Fatia vertical inicial

A primeira fatia funcional completa sera:

```text
payload padrao conhecido
  -> autenticacao
  -> persistencia idempotente
  -> modelo nfe/v1
  -> validacao do cenario piloto
  -> numeracao
  -> XML assinado
  -> autorizacao em homologacao
  -> XML e protocolo persistidos
  -> consulta com status authorized
  -> webhook final
```

Entradas customizadas, cobertura fiscal adicional e refinamentos somente ampliarao essa fatia depois que o fluxo principal estiver comprovado.

## Prioridades

| Prioridade | Significado |
| --- | --- |
| `P0` | Necessario para seguranca, integridade fiscal ou caminho critico do MVP. |
| `P1` | Necessario para operacao produtiva do piloto. |
| `P2` | Evolucao posterior que nao bloqueia o piloto inicial. |

Todo item dos Marcos 0 a 11 sera `P0` ou `P1`. Itens da secao de adiamentos serao `P2` ate nova decisao.

## Marco 0 - Recorte e prova tecnica

### Objetivo

Eliminar incertezas externas e provar os pontos tecnicos de maior risco antes de construir o produto completo.

### Backlog

#### `M0-01` - Preencher e aprovar o manifesto do piloto

Entregas:

- UF, regime, operacao, empresa, produtos e tributacao definidos;
- responsaveis fiscal e tecnico identificados;
- versao inicial do manifesto registrada;
- cenarios positivos e negativos selecionados.

#### `M0-02` - Disponibilizar certificado e credenciais de homologacao

Entregas:

- certificado A1 valido e correspondente ao emitente;
- cadeia e validade verificadas;
- segredo transmitido por canal seguro;
- procedimento temporario de acesso restrito;
- proibicao de adicionar PFX ou senha ao Git.

#### `M0-03` - Registrar pacotes e servicos oficiais aplicaveis

Entregas:

- XSD e versoes aplicaveis;
- Notas Tecnicas relevantes;
- autorizador por UF e ambiente;
- URLs, operacoes e protocolos esperados;
- hashes e origem dos pacotes preservados.

#### `M0-04` - Confirmar capacidades dos provedores iniciais

Entregas:

- PostgreSQL 18 suportado;
- armazenamento privado de objetos;
- mecanismo de criptografia envelope ou servico equivalente;
- backup com recuperacao ponto no tempo;
- ambiente separado de homologacao;
- destino de logs e metricas;
- limites, custos e responsabilidades registrados.

O processamento da outbox diretamente no PostgreSQL e suficiente para o MVP; broker externo nao e requisito deste marco.

#### `M0-05` - Provar XML, validacao e assinatura

Entregas:

- fixture fiscal revisada;
- XML deterministico;
- validacao no XSD selecionado;
- assinatura XML valida com A1;
- verificacao independente da assinatura;
- nenhuma persistencia insegura do certificado.

O codigo desta prova pode ser descartavel e nao sera promovido automaticamente para producao.

#### `M0-06` - Executar smoke test controlado na homologacao

Entregas:

- conectividade TLS confirmada;
- solicitacao assinada transmitida;
- resposta tecnica preservada;
- autorizacao ou rejeicao fiscal compreendida;
- limitacoes e ajustes registrados.

### Dependencias

- dados reais da empresa piloto;
- certificado A1;
- revisao fiscal do cenario;
- disponibilidade do ambiente de homologacao.

### Criterio de saida

O manifesto esta aprovado, o XML valida e assina corretamente e existe evidencia de comunicacao controlada com o autorizador de homologacao. Bloqueios externos remanescentes possuem responsavel e plano documentados.

## Marco 1 - Fundacao de engenharia

### Objetivo

Criar uma base Go reproduzivel, testavel, observavel e segura para API e worker.

### Backlog

#### `M1-01` - Estruturar o repositorio

- `/cmd/api`;
- `/cmd/worker`;
- pacotes internos por responsabilidade definida no planejamento;
- separacao entre dominio, aplicacao, infraestrutura e adaptadores;
- remocao do template inicial do GoLand.

#### `M1-02` - Configuracao por ambiente

- configuracao tipada;
- validacao no startup;
- segredos fora do codigo;
- desenvolvimento, CI, homologacao e producao separados;
- falha segura quando configuracao critica estiver ausente.

#### `M1-03` - Ciclo de vida dos processos

- inicializacao e encerramento gracioso;
- timeouts;
- health, readiness e liveness;
- tratamento central de panic;
- identificacao de versao e build.

#### `M1-04` - Qualidade automatizada

- build reproduzivel;
- testes unitarios e de integracao;
- `go test`, `go vet` e formatacao no CI;
- analise de dependencias e segredos;
- cobertura informativa, sem substituir testes de risco;
- artefatos de teste preservados quando houver falha.

#### `M1-05` - Observabilidade basica

- logs estruturados;
- `request_id`, `correlation_id` e `trace_id`;
- mascaramento centralizado;
- metricas de processo e HTTP;
- nenhum payload ou segredo integral.

#### `M1-06` - Contratos e fixtures iniciais

- OpenAPI inicial dos endpoints definidos;
- catalogo inicial de erros;
- fixtures versionadas e sem dados produtivos;
- relogio, IDs e dependencias externas injetaveis para testes.

### Criterio de saida

API e worker iniciam, encerram, expõem saude, produzem logs seguros e passam no pipeline automatizado. Nao existe dependencia de maquina pessoal para executar os testes basicos.

## Marco 2 - Banco e multi-tenancy

### Objetivo

Implementar o modelo fisico, migrations e isolamento de tenants definidos na Etapa 9.

### Backlog

#### `M2-01` - Ferramenta e trilhas de migration

- Goose e comando proprio;
- trilhas `platform` e `tenant`;
- lock global;
- historico por tenant;
- execucao em lote e individual;
- retomada segura e deteccao de drift.

#### `M2-02` - Schema central

- grupos;
- API keys;
- atores administrativos;
- auditoria central;
- controle de provisionamento e migrations.

#### `M2-03` - Schema de tenant

- empresas e integracoes;
- configuracoes, produtos e destinatarios;
- requests, documentos, itens e snapshots;
- numeracao, tentativas, transicoes e eventos;
- certificados e artefatos;
- outbox, leases e tabelas operacionais necessarias.

#### `M2-04` - Provisionamento idempotente

- criacao de schema validado;
- migrations ate a versao atual;
- estado `provisioning`, `active` ou `failed` coerente;
- nenhuma ativacao parcial;
- retomada depois de falha.

#### `M2-05` - Papeis e contexto transacional

- credenciais separadas de runtime, migration e operacao;
- menor privilegio;
- `SET LOCAL search_path` somente em transacao;
- runtime sem DDL;
- nomes de schema nunca originados do payload.

#### `M2-06` - Concorrencia e integridade

- constraints e indices essenciais;
- UUID v7 gerado pela aplicacao;
- valores fiscais em `numeric`;
- status por `text` e checks;
- historicos imutaveis;
- outbox na mesma transacao do efeito de negocio.

#### `M2-07` - Testes de banco

- tenant A sem acesso ao tenant B;
- provisionamento concorrente;
- migration falha e retomada;
- novo tenant na versao atual;
- drift detectado;
- privilegios negados;
- restauracao em banco descartavel.

### Dependencias

- Marco 1;
- PostgreSQL 18 disponivel no desenvolvimento e CI.

### Criterio de saida

Um grupo pode ser provisionado de forma idempotente, operar somente em seu schema e evoluir por migration sem afetar indevidamente os demais tenants.

## Marco 3 - Autenticacao e configuracao segura

### Objetivo

Permitir que integracoes e operadores acessem somente grupos, empresas e recursos autorizados, preservando segredos e auditoria.

### Backlog

#### `M3-01` - Ciclo de vida de API keys

- geracao criptograficamente segura;
- prefixo identificavel e segredo exibido uma vez;
- hash resistente;
- validacao em tempo constante;
- ambiente, status, expiracao e ultimo uso;
- rotacao, janela de sobreposicao e revogacao;
- rate limit por credencial e grupo.

#### `M3-02` - Middleware de autenticacao e tenant

- autenticacao antes de resolver recurso;
- contexto de grupo confiavel;
- autorizacao de empresa e integracao;
- respostas que nao revelam outro tenant;
- correlacao e auditoria de falhas relevantes.

#### `M3-03` - Administracao inicial

- comandos ou API interna para grupo, empresa, integracao e key;
- perfis administrativos da Etapa 8;
- justificativa para acoes criticas;
- auditoria separada dos logs.

#### `M3-04` - Certificado A1

- upload limitado e validado;
- criptografia envelope;
- senha protegida;
- metadados, validade e status;
- descriptografia somente pelo worker autorizado;
- substituicao, bloqueio e revogacao auditados;
- ausencia de PFX em log, fila e arquivo temporario persistente.

#### `M3-05` - Testes de seguranca do marco

- chave invalida, revogada, expirada e de outro ambiente;
- rotacao concorrente;
- acesso cruzado por IDs validos de outro tenant;
- PFX alterado ou senha incorreta;
- identidade sem permissao de descriptografia;
- tentativa de registrar segredo.

### Criterio de saida

Uma integracao autenticada acessa apenas seu grupo e empresas permitidas; certificados permanecem cifrados e somente o fluxo autorizado consegue usa-los.

## Marco 4 - Nucleo fiscal offline

### Objetivo

Converter dados comerciais em um modelo `nfe/v1` completo, rastreavel e validado para o manifesto piloto sem depender da SEFAZ.

### Backlog

#### `M4-01` - Tipos e contratos canonicos

- envelope do documento;
- emitente, destinatario, itens, totais, transporte e pagamentos;
- tipos de documento, decimal, data, unidade e codigos fiscais;
- versao do contrato preservada.

#### `M4-02` - Normalizacao

- CPF, CNPJ, CEP e inscricoes;
- datas, casas decimais e arredondamento;
- textos, unidades e codigos;
- erros com caminho do campo original e canonico.

#### `M4-03` - Cadastros e enriquecimento

- empresa;
- produto;
- destinatario quando aplicavel;
- regras fiscais versionadas;
- precedencia e origem de cada campo;
- proibicao de inferencia ambigua.

#### `M4-04` - Matriz `nfe/v1`

- campos sempre obrigatorios;
- blocos condicionais;
- ausencia, ambiguidade e conflito;
- totais e arredondamentos;
- rastreabilidade de regras aplicadas.

#### `M4-05` - Pacote do cenario piloto

- manifesto versionado;
- CFOP, CST ou CSOSN e aliquotas aplicaveis;
- regras de habilitacao por ambiente;
- versoes oficiais associadas;
- aprovacao fiscal registrada.

#### `M4-06` - Fixtures e testes fiscais

- caso minimo autorizado esperado;
- caso completo do recorte;
- campos ausentes;
- conflito de precedencia;
- totais divergentes;
- produto ou regra inexistente;
- cenario nao habilitado;
- testes de propriedade para valores e arredondamento onde aplicavel.

### Dependencias

- Marcos 1 a 3;
- manifesto do Marco 0 aprovado.

### Criterio de saida

Fixtures do cenario piloto produzem de forma deterministica um snapshot fiscal completo ou erros estruturados e acionaveis. Nenhum campo obrigatorio e preenchido por suposicao silenciosa.

## Marco 5 - Ingestao, mapeamento e idempotencia

### Objetivo

Receber payloads padrao e customizados, preservar a intencao original e criar um documento consultavel sem duplicidade.

### Backlog

#### `M5-01` - Contrato padrao

- `POST /v1/fiscal-documents`;
- validacao estrutural;
- `Idempotency-Key` obrigatoria;
- `202 Accepted` e `Location`;
- limites de corpo, profundidade e tempo;
- payload original tratado como nao confiavel.

#### `M5-02` - Validacao sem emissao

- endpoint ou operacao definida na Etapa 6;
- mapping, normalizacao e validacao;
- nenhuma numeracao, tentativa ou transmissao;
- versoes e erros devolvidos.

#### `M5-03` - Entrada customizada

- endpoint por `integration_id`;
- mapping declarativo e versionado;
- allowlist de transformacoes;
- adaptador especifico somente por processo controlado;
- nenhuma regra fiscal arbitraria no mapping.

#### `M5-04` - Persistencia e rastreabilidade

- raw payload conforme politica de retencao;
- payload normalizado;
- versoes do contrato, mapping e regras;
- fingerprint canonica;
- request, documento e outbox na mesma transacao;
- `external_reference`, `request_id` e `correlation_id`.

#### `M5-05` - Comportamento idempotente

- mesma key e fingerprint retornam o mesmo recurso;
- mesma key e conteudo diferente retornam `409`;
- concorrencia nao cria documento duplicado;
- repeticao nao reserva novo numero nem inicia fluxo paralelo.

#### `M5-06` - Consulta inicial

- `GET /v1/fiscal-documents/{id}`;
- status publico;
- isolamento por tenant;
- erro publico estruturado;
- links somente quando disponiveis.

### Criterio de saida

Uma requisicao valida cria exatamente um documento `pending`, consultavel e correlacionavel. Repeticoes e concorrencia obedecem a idempotencia sem efeito fiscal duplicado.

## Marco 6 - Estado, numeracao e processamento assincrono

### Objetivo

Executar o fluxo de emissao com estado consistente, numeracao atomica, concorrencia controlada e recuperacao segura.

### Backlog

#### `M6-01` - Maquina de estados

- estados internos e status publicos;
- transicoes permitidas;
- estado e historico na mesma transacao;
- transicao invalida recusada;
- motivo, ator, versao e horario preservados.

#### `M6-02` - Numeracao

- escopo por empresa, modelo, serie e ambiente;
- reserva atomica;
- concorrencia sem duplicidade;
- numero nunca reutilizado automaticamente;
- ajuste administrativo protegido e auditado.

#### `M6-03` - Outbox e consumidor

- publicacao idempotente;
- processamento direto da tabela no MVP;
- ordenacao apenas onde necessaria;
- retencao e limpeza segura;
- recuperacao depois de interrupcao.

#### `M6-04` - Leases e workers

- uma tentativa ativa por documento;
- aquisicao, renovacao e expiracao;
- worker interrompido antes e depois de efeitos externos;
- nenhum `group_id` ou schema arbitrario na mensagem;
- concorrencia testada com varios workers.

#### `M6-05` - Retentativas

- classificacao temporaria ou definitiva;
- backoff exponencial e jitter;
- limite por categoria;
- `processing_failed` depois do esgotamento;
- intervencao manual cria nova tentativa imutavel.

#### `M6-06` - Resultado desconhecido

- `authorization_unknown`;
- reconciliacao obrigatoria;
- proibicao de retransmitir antes de resultado seguro;
- desfecho para autorizacao, rejeicao, nova tentativa ou falha.

### Dependencias

- Marcos 2, 4 e 5.

### Criterio de saida

O worker processa documentos concorrentes sem duplicar numero ou tentativa, recupera interrupcoes e preserva estado suficiente para reconciliar qualquer efeito externo incerto.

## Marco 7 - XML, validacao e assinatura

### Objetivo

Produzir o XML fiscal do cenario piloto de forma deterministica, valida e assinada, pronto para transmissao.

### Backlog

#### `M7-01` - Pacotes oficiais versionados

- XSD e dependencias armazenados com origem e hash;
- versao associada ao cenario;
- atualizacao controlada;
- validacao bloqueada para pacote desconhecido.

#### `M7-02` - Geracao de XML

- namespaces, ordem, formatos e cardinalidades;
- identidade fiscal e chave de acesso;
- arredondamento coerente com snapshot;
- ausencia de dependencia do payload original;
- resultado deterministico para o mesmo snapshot e identidade.

#### `M7-03` - Validacao

- schema antes da assinatura quando aplicavel;
- schema depois da montagem final;
- erros traduzidos para diagnostico seguro;
- XML invalido nunca transmitido.

#### `M7-04` - Assinatura A1

- acesso restrito e temporario ao material descriptografado;
- algoritmo e referencias aplicaveis;
- cadeia e validade verificadas;
- assinatura verificada localmente;
- segredo ausente de log, trace, fila e arquivo persistente.

#### `M7-05` - Persistencia previa e integridade

- XML vinculado a tentativa e versoes;
- hash calculado;
- artefato privado;
- identidade preservada antes de qualquer chamada externa;
- impossibilidade de trocar silenciosamente XML da tentativa.

#### `M7-06` - Testes

- fixtures minimas e completas;
- assinatura alterada;
- certificado invalido, vencido ou divergente;
- schema incompativel;
- atualizacao de pacote;
- concorrencia e repeticao deterministica.

### Criterio de saida

Um snapshot aprovado gera XML validado e assinatura verificavel, preservado com hash e vinculo imutavel a tentativa, certificado e versoes utilizadas.

## Marco 8 - Integracao e homologacao SEFAZ

### Objetivo

Autorizar e reconciliar NF-e do cenario piloto no ambiente real de homologacao.

### Backlog

#### `M8-01` - Catalogo de servicos

- autorizador por UF e ambiente;
- servico, versao, SOAP action e endpoint;
- TLS, timeout e limite de resposta;
- nenhum endpoint vindo do payload;
- atualizacao versionada.

#### `M8-02` - Cliente de autorizacao

- envio individual ou lote conforme decisao do modulo;
- resposta definitiva;
- recibo e consulta posterior;
- codigos e motivos preservados;
- transporte separado da interpretacao fiscal.

#### `M8-03` - Reconciliacao

- timeout antes, durante e depois do envio;
- resposta incompleta;
- consulta por recibo, protocolo ou chave quando aplicavel;
- nenhuma retransmissao cega;
- evidencia de cada tentativa.

#### `M8-04` - Persistencia do resultado

- protocolo e resposta relevante;
- XML autorizado;
- hash e referencia de armazenamento;
- transicao atomica;
- `authorized` somente depois de persistencia segura;
- rejeicao fiscal separada de erro tecnico.

#### `M8-05` - Homologacao do manifesto

- caso autorizado;
- casos rejeitados conhecidos;
- consulta de recibo;
- consulta pela chave ou protocolo;
- indisponibilidade e timeout simulados;
- evidencias, versoes e limitacoes registradas;
- aprovacao tecnica e fiscal.

#### `M8-06` - Consulta e XML publicos

- status final;
- chave e protocolo autorizados;
- download protegido;
- auditoria de acesso;
- recurso de outro tenant indistinguivel de inexistente.

### Dependencias

- Marcos 0 a 7;
- homologacao SEFAZ disponivel;
- certificado e empresa habilitados.

### Criterio de saida

Ao menos uma NF-e do manifesto e autorizada de ponta a ponta em homologacao, com protocolo e XML preservados, consulta publica coerente e evidencia fiscal aprovada. Rejeicao e resultado desconhecido tambem possuem comportamento demonstrado.

## Marco 9 - Eventos e artefatos fiscais

### Objetivo

Completar o ciclo fiscal inicial com cancelamento, inutilizacao e gestao segura dos artefatos.

### Backlog

#### `M9-01` - Cancelamento

- pre-condicoes e prazo aplicavel;
- justificativa;
- idempotencia;
- XML assinado do evento;
- autorizacao, rejeicao e resultado desconhecido;
- status `cancelled` somente depois da confirmacao.

#### `M9-02` - Inutilizacao

- empresa, ambiente, modelo, serie e faixa;
- numeros elegiveis;
- justificativa;
- idempotencia;
- assinatura, protocolo e auditoria;
- numero nunca reutilizado.

#### `M9-03` - Ciclo de vida dos artefatos

- XML assinado, autorizado e de eventos;
- metadados, hashes e referencias;
- acesso privado e URLs temporarias quando usadas;
- retencao, legal hold e descarte;
- auditoria de acesso e download.

#### `M9-04` - Testes e homologacao

- cancelamento autorizado e rejeitado;
- inutilizacao valida e invalida;
- timeout com reconciliacao;
- integridade dos artefatos;
- separacao entre ambientes e tenants.

### Criterio de saida

Cancelamento e inutilizacao do recorte inicial funcionam em homologacao, preservam protocolo e historico e nao permitem alteracao insegura da identidade fiscal.

## Marco 10 - Operacao, webhooks e suporte

### Objetivo

Tornar o fluxo observavel, notificavel e operavel sem acesso manual direto ao banco.

### Backlog

#### `M10-01` - Correlacao e erros

- propagacao de todos os IDs definidos na Etapa 10;
- catalogo de erros publicado;
- categoria, retentabilidade e orientacao;
- codigos estaveis;
- ausencia de detalhe sensivel.

#### `M10-02` - Webhooks

- endpoint por integracao e ambiente;
- segredo cifrado e rotacao;
- eventos finais do MVP;
- HMAC-SHA256 e protecao contra replay;
- entrega pelo menos uma vez;
- retentativas, degradacao, suspensao e reenvio;
- SSRF, DNS rebinding, timeout e limite de resposta;
- historico e auditoria.

#### `M10-03` - Metricas e traces

- API e ingestao;
- fila, outbox, leases e workers;
- SEFAZ e reconciliacao;
- banco, armazenamento e migrations;
- webhooks;
- seguranca;
- limites de cardinalidade e privacidade.

#### `M10-04` - Dashboards e alertas

- fluxo fiscal;
- API;
- workers e fila;
- SEFAZ;
- webhooks;
- infraestrutura e seguranca;
- alertas acionaveis com responsavel e runbook.

#### `M10-05` - Operacoes administrativas

- localizar documento;
- consultar tentativas e transicoes;
- solicitar reconciliacao;
- reprocessar quando permitido;
- reenviar webhook;
- suspender integracao ou credencial;
- verificar certificado, migration e drift;
- justificativa, autorizacao e auditoria.

#### `M10-06` - Runbooks e atendimento

- runbooks obrigatorios da Etapa 10;
- severidades `P1` a `P4`;
- horario, canal, titulares e substitutos;
- simulacao de alertas e incidentes;
- evidencias de recuperacao.

#### `M10-07` - Baseline operacional

- disponibilidade e latencia de ingestao;
- tempo interno separado da SEFAZ;
- capacidade inicial dos workers;
- tempo de primeira entrega de webhook;
- limiares ajustados com dados de homologacao.

### Criterio de saida

Uma operacao consegue acompanhar, diagnosticar, reconciliar e notificar o fluxo por interfaces protegidas. Alertas e runbooks foram exercitados, e nenhum procedimento normal exige editar estado fiscal no banco.

## Marco 11 - Prontidao e piloto controlado

### Objetivo

Comprovar que o recorte homologado pode operar em producao com seguranca, recuperacao e suporte proporcionais ao risco.

### Backlog

#### `M11-01` - Testes de sistema

- fluxo completo autorizado e rejeitado;
- concorrencia e idempotencia;
- falhas de rede, banco, storage e worker;
- recuperacao depois de interrupcao;
- carga compativel com o piloto;
- soak test do processamento assincrono;
- regressao de fixtures fiscais.

#### `M11-02` - Seguranca

- isolamento multi-tenant independente;
- autenticacao, autorizacao e rate limit;
- revisao de segredos e dependencias;
- testes de SSRF e destinos externos;
- protecao de certificado e XML;
- avaliacao independente sem vulnerabilidade critica ou alta aberta.

#### `M11-03` - Backup e recuperacao

- backup criptografado;
- restauracao em ambiente isolado;
- RPO e RTO medidos;
- integridade entre banco, XML e protocolo;
- procedimento e responsaveis registrados;
- producao nunca sobrescrita no teste.

#### `M11-04` - Revisoes externas obrigatorias

- revisao fiscal do manifesto, regras e retencao;
- revisao juridica de papeis, contratos, incidentes e encerramento;
- termos de tratamento de dados;
- fornecedores e suboperadores registrados;
- responsaveis nominais por seguranca, privacidade, fiscal, credenciais e incidentes.

#### `M11-05` - Preparacao de release

- migrations testadas em copia representativa;
- estrategia expand/contract;
- configuracao e segredos por ambiente;
- rollback de aplicacao e roll-forward de banco;
- checklist e aprovadores;
- monitoramento reforcado depois do deploy.

#### `M11-06` - Plano do piloto

- um grupo, uma empresa e uma integracao iniciais;
- cenarios e limites habilitados;
- volume e janela controlados;
- criterios de interrupcao;
- procedimento de suporte e comunicacao;
- reconciliacao e verificacao diaria;
- periodo de observacao antes de ampliar cobertura.

#### `M11-07` - Execucao do piloto

- ativacao somente depois de todos os portoes;
- acompanhamento das primeiras emissoes;
- comparacao entre ERP, plataforma, XML e SEFAZ;
- incidentes e desvios registrados;
- revisao tecnica, fiscal e operacional;
- decisao explicita de continuar, corrigir ou interromper.

### Criterio de saida

O piloto opera dentro do manifesto e dos limites aprovados, sem falha critica aberta, com restauracao comprovada, suporte definido, resultados fiscais conciliados e decisao registrada sobre a proxima expansao.

## Dependencias criticas

| Dependencia | Necessaria antes de | Responsabilidade |
| --- | --- | --- |
| Manifesto fiscal aprovado | M4 e M8 | Responsavel fiscal e produto. |
| Empresa e A1 de homologacao | M0, M7 e M8 | Empresa piloto e responsavel por credenciais. |
| PostgreSQL 18 | M2 | Infraestrutura. |
| Armazenamento privado | M7 e M8 | Infraestrutura e seguranca. |
| Criptografia envelope | M3 | Seguranca e infraestrutura. |
| Pacotes e endpoints oficiais | M0, M7 e M8 | Modulo fiscal. |
| Ambiente de homologacao | M0 e M8 | Infraestrutura e SEFAZ. |
| Revisoes fiscal e juridica | M11 | Especialistas responsaveis. |
| Avaliacao independente de seguranca | M11 | Avaliador sem conflito relevante. |
| Canal e cobertura de suporte | M11 | Operacao. |

Dependencia externa bloqueada devera possuir dono, evidencia, data de revisao e impacto. Nao sera escondida por mock nem marcada como concluida por conveniencia.

## Requisitos transversais

### Seguranca

Cada marco devera revisar:

- autenticacao e autorizacao;
- isolamento de tenant;
- classificacao dos dados;
- segredo e criptografia;
- logs e erros;
- auditoria;
- retencao;
- dependencias e superficie externa.

### Observabilidade

Cada capacidade nova devera adicionar:

- identificadores de correlacao;
- logs estruturados e mascarados;
- metricas de sucesso, falha e duracao;
- trace nos limites relevantes;
- alerta somente quando houver acao operacional;
- runbook quando introduzir falha nova de alto impacto.

### Documentacao

Cada entrega devera manter:

- contrato externo;
- decisao de dominio;
- migration e modelo de dados;
- catalogo de erro;
- configuracao e limites;
- testes e evidencias;
- runbook ou procedimento operacional.

### Compatibilidade

- API publica versionada;
- `nfe/v1` versionado separadamente;
- mapping e formulario com versoes proprias;
- regras e manifestos fiscais imutaveis depois do uso;
- migrations por expand/contract;
- consumidores de outbox e webhook idempotentes.

## Definition of Ready

Um item pode entrar em implementacao quando possui:

1. objetivo e valor claros;
2. documento de origem identificado;
3. dependencias atendidas ou explicitamente simulaveis;
4. escopo e fora de escopo;
5. contrato ou comportamento esperado;
6. criterios de aceite verificaveis;
7. dados e fixtures sem informacao produtiva indevida;
8. riscos de seguranca, fiscal e multi-tenancy avaliados;
9. estrategia de teste;
10. responsavel por aceitar o resultado quando exigir decisao externa.

Um item fiscal especifico nao esta pronto sem manifesto aplicavel. Um item que acessa SEFAZ nao esta pronto para homologacao sem certificado e empresa validos.

## Definition of Done

Uma entrega somente esta concluida quando, conforme o risco:

1. codigo implementado e revisado;
2. formatacao, build, vet e testes passam;
3. testes unitarios, integracao e contrato relevantes existem;
4. concorrencia e idempotencia foram testadas quando aplicaveis;
5. isolamento multi-tenant foi demonstrado;
6. erros publicos estao documentados e nao vazam detalhes;
7. logs, metricas e traces nao contem dados proibidos;
8. migration possui retomada ou recuperacao testada;
9. auditoria existe para operacao critica;
10. documentacao e exemplos foram atualizados;
11. observabilidade e runbook acompanham nova falha operacional relevante;
12. evidencia de homologacao foi preservada quando envolve SEFAZ;
13. responsavel fiscal aprovou mudanca de regra ou cenario;
14. nenhum `TODO` critico ficou oculto no caminho executado;
15. criterio de aceite foi demonstrado em ambiente compativel.

Percentual de cobertura nao substitui esses requisitos. Codigo que funciona apenas no caminho feliz nao esta concluido.

## Estrategia de testes

### Unidade

- tipos e invariantes;
- normalizacao;
- regras e matriz fiscal;
- calculos e arredondamento;
- transicoes de estado;
- classificacao de erro;
- assinatura e verificacao isoladas;
- serializacao de contratos.

### Integracao

- PostgreSQL real descartavel;
- migrations e privilegios;
- transacoes, outbox, leases e concorrencia;
- armazenamento de objetos compativel;
- criptografia e certificado;
- cliente HTTP/SOAP com servidor controlado;
- entrega de webhook.

### Contrato

- OpenAPI e exemplos;
- compatibilidade de erros;
- versionamento de payload;
- webhooks e assinatura;
- respostas simuladas da SEFAZ baseadas em evidencias versionadas.

### Sistema

- API, worker, banco e storage juntos;
- fluxo autorizado, rejeitado e falho;
- interrupcao em pontos criticos;
- repeticao e concorrencia;
- recuperacao e reconciliacao;
- observabilidade e alertas.

### Homologacao fiscal

- empresa e certificado reais de homologacao;
- autorizador aplicavel;
- casos aprovados pelo manifesto;
- autorizacao, rejeicao, consulta e eventos;
- evidencias e versoes preservadas;
- aprovacao fiscal.

### Seguranca e resiliencia

- acesso cruzado entre tenants;
- credenciais e permissoes;
- segredo em log, trace, fila e temporario;
- SSRF e resposta externa maliciosa;
- carga e exaustao de recursos;
- backup, restauracao e integridade;
- simulacao de incidente.

## Ambientes

### Desenvolvimento

- dados sinteticos;
- dependencias locais ou descartaveis;
- sem certificado ou dado produtivo;
- mocks controlados para SEFAZ;
- mesmas versoes major de banco e runtime.

### CI

- ambiente efemero;
- migrations desde zero;
- testes paralelos com isolamento;
- fixtures versionadas;
- varreduras automatizadas;
- nenhum segredo de producao.

### Homologacao

- infraestrutura separada;
- credenciais e A1 proprios;
- endpoints oficiais de homologacao;
- storage e criptografia equivalentes ao desenho produtivo;
- observabilidade ativa;
- evidencia fiscal preservada.

### Producao

- acesso minimo;
- segredos exclusivos;
- backups e restauracao testados;
- alertas, canal e responsaveis ativos;
- somente manifestos e cenarios explicitamente habilitados;
- ativacao por portao e auditoria.

## Portoes

### `G0` - Pronto para implementar

- plano aprovado;
- backlog ordenado;
- ambiente de desenvolvimento e responsaveis definidos;
- bloqueios externos do M0 visiveis.

### `G1` - Pronto para integrar a fatia vertical

- M1 a M4 concluidos;
- tenant, autenticacao e certificado protegidos;
- modelo fiscal e fixture aprovados;
- CI verde.

### `G2` - Pronto para homologar na SEFAZ

- M5 a M7 concluidos;
- XML valida e assina;
- idempotencia, numeracao e resultado desconhecido testados;
- ambiente, empresa e certificado confirmados;
- nenhuma vulnerabilidade critica conhecida no caminho.

### `G3` - Pronto para preparar producao

- M8 a M10 concluidos;
- autorizacao, rejeicao, consulta, cancelamento e inutilizacao homologados;
- XML e protocolo preservados;
- alertas e runbooks exercitados;
- operacao nao depende de edicao manual no banco.

### `G4` - Pronto para piloto produtivo

- testes e revisoes do M11 aprovados;
- restauracao comprovada;
- revisoes fiscal, juridica e independente de seguranca registradas;
- cobertura e canal de suporte ativos;
- checklist, responsaveis, limites e criterio de interrupcao aprovados;
- nenhuma pendencia critica ou alta aberta.

Falhar em um portao impede avancar ao ambiente seguinte, mas nao impede corrigir e reapresentar as evidencias.

## Estrategia de release

- integracao continua em branch principal protegida;
- mudancas pequenas e reversiveis;
- feature flags ou habilitacoes para modulo, ambiente e cenario;
- deploy da aplicacao separado da ativacao fiscal;
- migrations expand/contract;
- preferencia por roll-forward do banco;
- rollback da aplicacao testado;
- ativacao gradual por grupo e empresa;
- monitoramento reforcado depois de migration, deploy e habilitacao;
- release nao amplia automaticamente UF, regime ou cenario.

## Modelo de item do backlog

Cada item detalhado durante a execucao usara:

```text
ID:
Titulo:
Prioridade:
Objetivo:
Documentos de origem:
Dependencias:
Escopo:
Fora do escopo:
Entregaveis:
Criterios de aceite:
Testes obrigatorios:
Observabilidade:
Seguranca e dados:
Riscos:
Evidencias:
Responsavel pela aceitacao:
```

Exemplo resumido:

```text
ID: M5-05
Titulo: Idempotencia da solicitacao fiscal
Prioridade: P0

Criterios de aceite:
- mesma chave e mesmo payload retornam o mesmo documento;
- mesma chave e payload diferente retornam 409;
- concorrencia nao cria dois documentos;
- nenhuma repeticao reserva outro numero;
- o fluxo permanece correlacionavel e auditavel.
```

## Riscos principais

| Risco | Impacto | Tratamento |
| --- | --- | --- |
| Cenario fiscal amplo ou indefinido | Retrabalho e emissao incorreta. | Manifesto estreito e aprovacao fiscal no M0. |
| Certificado ou empresa indisponivel | Homologacao bloqueada. | Confirmar antes do nucleo especifico e manter dono externo. |
| XML aceito localmente e recusado pela SEFAZ | Atraso no caminho critico. | Spike M0 e homologacao incremental. |
| Duplicidade por timeout | Risco fiscal grave. | Idempotencia, identidade persistida e reconciliacao obrigatoria. |
| Vazamento entre tenants | Incidente critico. | Contexto transacional, privilegios e testes independentes. |
| PFX ou segredo em log | Comprometimento de credencial. | Biblioteca central, testes e varredura automatica. |
| Migration falha em parte dos tenants | Versoes divergentes. | Lock, historico, lotes, retomada e drift. |
| Perda de XML autorizado | Risco fiscal e operacional. | Persistencia antes de `authorized`, hash, backup e restauracao. |
| Dependencia excessiva de mock | Falsa confianca. | Portoes exigem homologacao real. |
| Observabilidade implementada tarde | Falha impossivel de diagnosticar. | Requisito transversal desde M1. |
| Sobrecarga de uma equipe pequena | Trabalho incompleto em muitos marcos. | Limite de WIP e caminho critico sequencial. |
| Promessa de SLA externo | Compromisso fora do controle. | Separar tempo interno, SEFAZ e cliente. |

O registro de riscos sera atualizado a cada portao com probabilidade, impacto, responsavel e evidencia de mitigacao.

## Matriz de rastreabilidade

| Planejamento de origem | Marcos principais |
| --- | --- |
| Fundacao de flexibilidade e conformidade | M0, M4, M5, M11 |
| Dominio e nomenclatura | M1, M2, M4, M5 |
| Modelo interno de documentos fiscais | M4, M5, M7 |
| Regras e matriz de preenchimento | M0, M4, M7, M8 |
| Numeracao, idempotencia e estados | M5, M6, M8, M9 |
| Contratos e mapping | M1, M4, M5, M10 |
| Integracao com a SEFAZ | M0, M6, M7, M8, M9 |
| Seguranca e dados sensiveis | Todos, especialmente M3, M10 e M11 |
| Banco e migrations multi-tenant | M2, M5, M6, M11 |
| Operacao e suporte | Todos desde M1, completado em M10 e M11 |

Nenhum marco substitui seu documento de origem. Este plano ordena a execucao; detalhes normativos permanecem nos planejamentos especializados.

## Checklist anterior ao primeiro cliente em producao

### Fiscal

- manifesto aprovado e versionado;
- cenario exercitado em homologacao;
- autorizacao, rejeicao, consulta e eventos evidenciados;
- pacotes oficiais e limitacoes registrados;
- revisao fiscal concluida;
- somente cenario homologado habilitado.

### Seguranca e privacidade

- papeis contratuais revisados;
- fornecedores registrados;
- API keys e certificados protegidos;
- isolamento testado;
- dados sensiveis ausentes da observabilidade;
- avaliacao independente sem alta ou critica aberta;
- responsaveis e fluxo de incidente definidos.

### Dados e recuperacao

- migrations aplicadas e sem drift;
- backup bem-sucedido;
- restauracao testada;
- XML, protocolo, hash e referencia conciliados;
- RPO e RTO medidos;
- retencao e descarte configurados.

### Operacao

- dashboards e alertas ativos;
- runbooks acessiveis e exercitados;
- canal, cobertura, titulares e substitutos definidos;
- webhooks testados;
- operacoes administrativas auditadas;
- criterio de interrupcao do piloto aprovado.

### Release

- build identificado e reproduzivel;
- CI verde;
- configuracao revisada;
- segredos do ambiente corretos;
- migrations e deploy ensaiados;
- rollback e roll-forward compreendidos;
- aprovacao `G4` registrada.

## Criterios para concluir a implementacao do MVP

O MVP somente podera ser considerado implementado quando:

1. Marcos 0 a 11 atenderem seus criterios de saida;
2. todos os portoes estiverem aprovados;
3. a fatia vertical estiver demonstrada em homologacao e no piloto controlado;
4. idempotencia e concorrencia nao permitirem duplicidade;
5. resultado desconhecido for reconciliado antes de retransmitir;
6. isolamento multi-tenant estiver testado;
7. XML e protocolo autorizados estiverem integros e recuperaveis;
8. operacao, alertas, runbooks e suporte estiverem ativos;
9. revisoes fiscal, juridica e de seguranca estiverem registradas;
10. nao houver vulnerabilidade critica ou alta aberta;
11. nao houver pendencia fiscal critica no manifesto habilitado;
12. documentos das capacidades entregues puderem ser movidos para `docs/implementado` com suas evidencias.

## Decisoes adiadas conscientemente

- ampliacao para outras UFs e regimes;
- NFC-e e outros documentos fiscais;
- contingencia em producao;
- DANFE;
- carta de correcao;
- broker externo;
- microsservicos separados;
- Kubernetes ou orquestracao equivalente;
- operacao multi-regiao;
- painel administrativo completo;
- portal self-service;
- multiplos webhooks por integracao;
- data warehouse e analytics;
- HSM dedicado;
- SIEM e SOC dedicados;
- banco dedicado por tenant;
- particionamento e sharding;
- SLA contratual por autorizador externo;
- suporte 24x7 antes de existir plantao real.

Esses itens somente entrarao no caminho critico por requisito legal, risco, contrato ou evidencia de volume. Sua ausencia nao reduz os controles essenciais do MVP.

## Decisoes consolidadas da Etapa 11

1. A implementacao seguira doze marcos, de M0 a M11.
2. O Marco 0 reduz riscos externos antes da construcao completa.
3. UF, regime, operacao e empresa piloto serao dados reais de um manifesto versionado, nunca suposicoes do codigo.
4. A primeira entrega fiscal sera NF-e modelo 55 em emissao normal e homologacao.
5. A arquitetura sera multi-tenant mesmo que o piloto comece com um grupo.
6. O caminho principal sera uma fatia vertical autorizada de ponta a ponta.
7. Banco e isolamento precedem persistencia de documentos reais.
8. Nucleo fiscal offline precede geracao e transmissao de XML.
9. Idempotencia, numeracao e reconciliacao precedem qualquer emissao completa.
10. Broker externo nao sera obrigatorio; a outbox PostgreSQL atende ao MVP.
11. Observabilidade, seguranca e testes atravessam todos os marcos.
12. Homologacao real sera exigida; mocks nao concluem integracao fiscal.
13. Cancelamento e inutilizacao pertencem ao MVP; contingencia produtiva fica adiada.
14. Webhooks e operacoes administrativas entram antes do piloto.
15. O painel visual nao bloqueia a operacao produtiva.
16. Revisoes fiscal, juridica e independente de seguranca sao portoes de producao.
17. Restauracao comprovada e requisito, nao tarefa posterior.
18. Progresso sera medido por criterios demonstrados, nao por percentual subjetivo.
19. O piloto comecara limitado e exigira decisao explicita antes de expandir.
20. Cada capacidade implementada preservara rastreabilidade ate seu planejamento de origem.

## Resultado esperado

Ao executar este plano, o projeto evoluira de documentacao arquitetural para uma plataforma capaz de receber, validar, emitir, consultar, cancelar e auditar NF-e do cenario piloto com isolamento multi-tenant, seguranca, recuperacao e operacao verificaveis.

A conclusao desta Etapa 11 encerra o ciclo de planejamento inicial. O trabalho seguinte comeca no Marco 0, preenchendo o manifesto real do piloto e provando XML, assinatura e comunicacao de homologacao antes de ampliar a implementacao.
