# Operacao e suporte

## Objetivo

Definir como a plataforma sera observada, operada e suportada durante todo o ciclo de emissao fiscal, desde o recebimento da solicitacao ate a autorizacao, rejeicao, falha, cancelamento ou reconciliacao.

A operacao deve conseguir responder, sem consultar diretamente o banco ou expor dados sensiveis:

- qual solicitacao originou o documento;
- em qual estado publico e interno ele se encontra;
- qual tentativa esta ou esteve em execucao;
- se a SEFAZ recebeu ou pode ter recebido a transmissao;
- qual erro exige correcao do cliente e qual permite retentativa;
- se o XML e o protocolo foram persistidos;
- quais notificacoes foram entregues ao cliente;
- quais acoes automaticas ou administrativas ocorreram.

## Principios

- A API de consulta e a fonte de verdade para o cliente; webhook e uma notificacao.
- Toda operacao relevante deve ser correlacionavel de ponta a ponta.
- Estado publico simples e estado interno detalhado permanecem separados.
- Logs, metricas, traces, historico fiscal e auditoria possuem finalidades diferentes.
- Nenhuma observabilidade pode expor segredo, XML completo, payload completo ou dados de outro tenant.
- Alertas devem representar uma condicao acionavel e apontar para um procedimento.
- Reprocessamento, reconciliacao e reenvio nunca podem apagar ou reescrever o historico.
- Operacao administrativa ocorre por comandos ou APIs controladas, nunca por alteracao manual direta no banco.
- O MVP tera capacidade operacional antes de possuir um painel administrativo completo.
- Dependencias externas, especialmente a SEFAZ e o destino de webhook do cliente, serao medidas separadamente da saude interna da plataforma.

## Escopo operacional do MVP

O primeiro MVP devera incluir:

- consulta de status e resultado do documento;
- identificadores de correlacao em API, worker, SEFAZ, armazenamento e webhook;
- erros publicos estruturados e catalogados;
- webhooks assinados para eventos finais relevantes;
- logs estruturados e mascarados;
- metricas de API, fila, worker, SEFAZ, persistencia e webhooks;
- traces distribuidos nos limites internos relevantes;
- dashboards tecnicos essenciais;
- alertas acionaveis;
- historico operacional e auditoria separados;
- runbooks para as falhas principais;
- operacoes administrativas minimas, protegidas e auditadas;
- classificacao de severidade e fluxo de atendimento;
- testes de falha, recuperacao, alertas e webhooks.

Nao fazem parte da primeira entrega:

- portal administrativo completo;
- central de atendimento propria;
- personalizacao irrestrita de dashboards por cliente;
- analytics fiscal ou comercial;
- garantia de ordenacao global de webhooks;
- promessa de tempo de autorizacao da SEFAZ;
- operacao ativa em multiplas regioes;
- automacao irrestrita de contingencia ou correcao fiscal.

## Modelo de correlacao

### Identificadores

Cada fluxo devera usar, conforme aplicavel:

| Identificador | Origem | Finalidade |
| --- | --- | --- |
| `request_id` | API | Identificar uma chamada HTTP individual. |
| `correlation_id` | Cliente ou API | Correlacionar o fluxo completo entre sistemas. |
| `external_reference` | Cliente | Relacionar o documento ao pedido ou registro do ERP. |
| `fiscal_document_request_id` | Plataforma | Identificar a intencao original e sua idempotencia. |
| `fiscal_document_id` | Plataforma | Identificar o documento fiscal durante todo o ciclo. |
| `issuance_attempt_id` | Plataforma | Identificar cada tentativa imutavel de emissao ou reconciliacao. |
| `outbox_event_id` | Plataforma | Identificar o evento interno publicado. |
| `webhook_event_id` | Plataforma | Permitir deduplicacao do evento pelo cliente. |
| `webhook_delivery_id` | Plataforma | Identificar cada tentativa de entrega do webhook. |
| `trace_id` e `span_id` | Observabilidade | Correlacionar execucao tecnica entre componentes. |

`request_id` sempre sera criado pela plataforma e devolvido na resposta. Um `correlation_id` valido recebido em cabecalho podera ser preservado; quando ausente, a plataforma criara um. Valores recebidos terao formato e tamanho limitados.

O valor integral da API key, do segredo de webhook e do certificado nunca sera usado como identificador. A chave de idempotencia somente podera aparecer em ferramenta operacional por representacao parcial ou hash nao reversivel.

### Propagacao

Os identificadores necessarios devem acompanhar:

```text
requisicao HTTP
  -> transacao e outbox
  -> fila ou consumidor da outbox
  -> worker e tentativa
  -> chamada e resposta da SEFAZ
  -> persistencia do artefato
  -> evento de webhook
  -> tentativa de entrega
```

Mensagens assincronas nao confiarao em `group_id`, schema ou empresa informados livremente pelo produtor. Esses dados serao resolvidos a partir de referencias internas persistidas e validadas.

## Consulta publica do documento

O endpoint definido na Etapa 6 permanece:

```http
GET /v1/fiscal-documents/{fiscal_document_id}
```

A resposta deve conter, conforme o estado e a permissao:

```json
{
  "id": "uuid",
  "status": "authorized",
  "external_reference": "PEDIDO-123",
  "document": {
    "model": "55",
    "series": 1,
    "number": 123,
    "access_key": "...",
    "protocol": "..."
  },
  "error": null,
  "timestamps": {
    "received_at": "2026-08-03T13:00:00Z",
    "processing_started_at": "2026-08-03T13:00:01Z",
    "finished_at": "2026-08-03T13:00:04Z"
  },
  "correlation_id": "uuid",
  "links": {
    "self": "/v1/fiscal-documents/uuid",
    "xml": "/v1/fiscal-documents/uuid/xml"
  }
}
```

Regras:

- somente status publicos definidos na maquina de estados serao expostos;
- links serao omitidos quando a operacao ou o artefato nao estiver disponivel;
- XML, protocolo e chave somente serao apresentados quando persistidos e autorizados para aquele tenant;
- estados internos, leases, detalhes de infraestrutura e stack traces nao serao publicos;
- o cliente podera consultar o recurso mesmo que um webhook tenha falhado;
- `404` nao revelara se o recurso existe em outro tenant;
- respostas poderao incluir `Retry-After` quando uma nova consulta imediata nao trouxer beneficio.

O status `authorized` somente sera publicado depois que protocolo e XML autorizado estiverem preservados conforme as Etapas 7, 8 e 9.

## Catalogo de erros

### Estrutura publica

O formato definido na Etapa 6 sera preservado e ampliado de forma compativel:

```json
{
  "code": "fiscal.field_required",
  "message": "O NCM do item 2 nao foi informado nem encontrado no cadastro.",
  "category": "validation",
  "field": "items[1].ncm",
  "source_field": null,
  "retryable": false,
  "request_id": "uuid",
  "correlation_id": "uuid",
  "details": {}
}
```

Campos novos poderao ser adicionados dentro da mesma versao, mas codigo, categoria e significado nao poderao mudar silenciosamente.

### Categorias

| Categoria | Significado | Retentativa automatica |
| --- | --- | --- |
| `request` | Requisicao, JSON ou cabecalho invalido. | Nao. |
| `authentication` | Credencial ausente, invalida ou inativa. | Nao sem corrigir credencial. |
| `authorization` | Identidade sem permissao para o recurso. | Nao. |
| `mapping` | Entrada customizada nao pode ser mapeada. | Nao sem corrigir dado ou configuracao. |
| `validation` | Modelo comercial ou fiscal invalido. | Nao sem correcao. |
| `configuration` | Cadastro, regra, certificado ou habilitacao ausente. | Nao sem intervencao. |
| `sefaz_rejection` | Autoridade fiscal rejeitou o documento. | Nao automaticamente. |
| `temporary_dependency` | Dependencia temporariamente indisponivel. | Sim, conforme politica. |
| `authorization_unknown` | Transmissao pode ter ocorrido e exige reconciliacao. | Nunca retransmitir antes de reconciliar. |
| `internal` | Falha inesperada da plataforma. | Somente quando classificada como segura. |

### Requisitos do catalogo

Cada codigo devera documentar:

- significado estavel;
- categoria;
- status HTTP, quando aplicavel;
- status publico resultante;
- campos que podem acompanhar o erro;
- se o cliente deve corrigir, aguardar, consultar ou acionar suporte;
- se a plataforma pode retentar;
- mensagem publica sem dado sensivel;
- referencia interna para runbook, quando operacional.

Codigos da SEFAZ serao preservados em campo proprio quando forem seguros e relevantes, mas nao substituirao o codigo estavel da plataforma.

## Webhooks

### Papel do webhook

Webhook informa uma mudanca relevante, mas nao substitui a consulta. A plataforma oferece entrega pelo menos uma vez; duplicidades sao possiveis e devem ser tratadas pelo `event_id`.

Nao havera garantia de ordenacao global. O consumidor devera usar `occurred_at`, status atual consultado e regras de transicao, sem assumir que a ordem de chegada representa a ordem definitiva do documento.

### Eventos do MVP

- `fiscal_document.authorized`;
- `fiscal_document.rejected`;
- `fiscal_document.failed`;
- `fiscal_document.cancelled`.

Eventos intermediarios poderao ser adicionados no futuro. O MVP evitara notificar cada estado interno para reduzir acoplamento e ruido operacional.

### Configuracao

No MVP, cada integracao e ambiente podera possuir um endpoint ativo, com:

```text
id
integration_id
environment
url
subscribed_events
status
secret_encrypted
secret_version
created_at
updated_at
disabled_at
disable_reason
```

Regras:

- producao aceitara somente HTTPS com certificado valido;
- ambiente de homologacao e producao terao configuracoes e segredos distintos;
- segredo sera gerado ou fornecido por fluxo protegido, armazenado cifrado e exibido somente na criacao ou rotacao;
- rotacao permitira sobreposicao padrao de 24 horas e maxima de sete dias entre duas versoes, sempre auditada;
- URL sera normalizada e validada no cadastro e novamente antes da conexao;
- destinos locais, privados, link-local, loopback, metadados de infraestrutura e portas proibidas serao bloqueados;
- redirecionamentos nao serao seguidos;
- resolucao DNS e endereco efetivo serao verificados para reduzir DNS rebinding;
- alteracao, teste, ativacao e desativacao exigirao permissao e auditoria.

### Envelope

```json
{
  "id": "evt_uuid",
  "type": "fiscal_document.authorized",
  "version": "1",
  "occurred_at": "2026-08-03T13:00:04Z",
  "data": {
    "fiscal_document_id": "uuid",
    "status": "authorized",
    "company_id": "uuid",
    "external_reference": "PEDIDO-123",
    "correlation_id": "uuid"
  },
  "links": {
    "fiscal_document": "/v1/fiscal-documents/uuid"
  }
}
```

O webhook nao transportara XML, payload original, certificado, segredo, stack trace ou detalhes internos. Campos fiscais adicionais somente serao incluidos por necessidade contratual, revisao de seguranca e versionamento.

### Assinatura e protecao contra replay

Cabecalhos minimos:

```http
X-API-NF-Event-Id: evt_uuid
X-API-NF-Delivery-Id: del_uuid
X-API-NF-Timestamp: 1785762000
X-API-NF-Signature: v1=hex_hmac_sha256
```

A assinatura sera calculada sobre o timestamp textual, um ponto e os bytes exatos do corpo:

```text
HMAC-SHA256(secret, timestamp + "." + raw_body)
```

O consumidor devera validar assinatura com comparacao em tempo constante, rejeitar timestamp com diferenca superior a cinco minutos em relacao ao seu relogio e deduplicar pelo `event_id`.

### Entrega e retentativas

- o evento de negocio e sua outbox serao gravados na mesma transacao;
- a publicacao e o consumidor serao idempotentes;
- cada tentativa criara um `webhook_delivery_id` proprio;
- qualquer resposta `2xx` concluira a entrega;
- cada chamada tera timeout total de dez segundos, limite de tres segundos para conexao e resposta lida ate 64 KiB;
- timeout, falha de rede, resposta acima do limite e resposta nao `2xx` serao registrados como falha;
- `429` e `5xx` serao considerados temporarios;
- `3xx` nao sera seguido;
- retentativas usarao backoff exponencial com jitter e limite configurado;
- a politica inicial sera de sete tentativas no total: a primeira imediata e as seguintes com intervalos-base de 1 minuto, 5 minutos, 30 minutos, 2 horas, 8 horas e 24 horas, aplicando jitter de ate 20%;
- `Retry-After` valido podera ser respeitado dentro dos limites da plataforma;
- depois do limite, a entrega ficara `exhausted` e gerara sinal operacional;
- tres eventos consecutivos com tentativas esgotadas marcarao o endpoint como `degraded` e gerarao alerta;
- dez eventos consecutivos esgotados, ou sete dias continuos sem entrega bem-sucedida, suspenderao novas tentativas do endpoint e exigirao reativacao explicita, sem descartar os eventos pendentes;
- reenvio manual criara nova entrega vinculada ao mesmo evento, sem alterar o evento original.

Uma entrega registra, no minimo:

```text
delivery_id
event_id
endpoint_id
attempt_number
status
scheduled_at
started_at
finished_at
http_status
failure_category
duration_ms
response_size
request_body_hash
secret_version
next_attempt_at
```

Corpo integral da resposta do cliente nao sera persistido por padrao. Uma amostra truncada e mascarada podera ser mantida por ate 30 dias quando necessaria ao diagnostico. Metadados e hashes de entrega permanecerao por um ano; a acao administrativa de reenvio permanecera na auditoria pelo prazo definido na Etapa 8.

### Teste de endpoint

A plataforma podera oferecer evento de teste que:

- use envelope e assinatura reais;
- seja identificado explicitamente como teste;
- passe pelas mesmas validacoes de destino;
- nao represente emissao fiscal;
- registre entrega e resultado;
- nao altere estado de documento.

## Logs estruturados

Logs de producao serao estruturados e terao, quando aplicavel:

```text
timestamp
level
service
environment
event_name
message
request_id
correlation_id
trace_id
fiscal_document_id
attempt_id
group_reference_controlled
company_reference_controlled
error_code
duration_ms
```

Regras:

- campos serao emitidos por biblioteca comum, nao montados livremente em cada modulo;
- identificadores de tenant somente serao incluidos em ambiente de acesso restrito e conforme necessidade operacional;
- CPF, CNPJ, e-mail, endereco, valores e campos livres serao mascarados;
- API key, autorizacao, PFX, senha, chave privada, segredo, XML e payload completos sao proibidos;
- stack trace podera existir somente em observabilidade interna protegida e nunca na resposta externa;
- nivel `debug` nao desabilitara protecoes em producao;
- erros esperados de validacao nao serao tratados como falha critica de infraestrutura;
- logs operacionais seguirao a retencao de 90 dias disponiveis e ate um ano em arquivo protegido definida na Etapa 8.

## Tracing

Tracing sera usado para diagnosticar limites e latencia, sem copiar conteudo fiscal para atributos ou eventos.

Spans recomendados:

- recebimento e persistencia da API;
- transacao e criacao da outbox;
- publicacao e consumo do evento;
- aquisicao e renovacao de lease;
- normalizacao, enriquecimento e validacao;
- geracao, validacao e assinatura do XML;
- chamada a SEFAZ e consulta de recibo;
- persistencia do protocolo e artefato;
- preparacao e entrega de webhook.

O contexto de trace sera propagado por mensagens assincronas, mas nao substituira os identificadores persistentes de negocio. Amostragem devera preservar erros e fluxos lentos sem exigir armazenamento de todos os traces em producao.

## Metricas

### API e ingestao

- requisicoes por rota normalizada, metodo, status e categoria de erro;
- latencia de recebimento e persistencia;
- solicitacoes aceitas e recusadas;
- conflitos de idempotencia;
- limites de uso e payload excedidos.

### Processamento assincrono

- profundidade e idade do item mais antigo da fila ou outbox;
- tempo entre recebimento, fila, inicio e conclusao;
- workers ativos e taxa de processamento;
- leases adquiridas, expiradas e recuperadas;
- documentos por estado publico e interno operacional;
- retentativas agendadas e esgotadas;
- documentos em `authorization_unknown` e tempo nesse estado;
- reconciliacoes pendentes, concluidas e falhas.

### SEFAZ e documentos

- chamadas por servico, autorizador e ambiente;
- duracao, timeout e falha de transporte;
- autorizacoes e rejeicoes por categoria segura;
- resultados desconhecidos;
- falhas de schema, assinatura e certificado;
- XML autorizado ou protocolo sem persistencia confirmada;
- cancelamentos e inutilizacoes por resultado.

### Webhooks

- eventos criados;
- entregas por resultado e classe HTTP;
- latencia ate a primeira tentativa e ate o sucesso;
- retentativas e entregas esgotadas;
- endpoints ativos, degradados e desativados;
- bloqueios de destino por seguranca.

### Banco, armazenamento e migrations

- disponibilidade e saturacao do pool;
- duracao e falhas de consultas essenciais;
- erros de transacao, lock e concorrencia;
- uso de armazenamento e falhas de acesso;
- hash ou artefato inconsistente;
- tenants atrasados, com migration falha ou drift;
- estado de backups e testes de restauracao.

### Seguranca

- falhas relevantes de autenticacao e autorizacao;
- tentativas de acesso entre tenants;
- operacoes administrativas por resultado;
- certificados proximos do vencimento;
- falhas de descriptografia ou acesso ao cofre;
- violacoes da politica de destino externo.

### Cardinalidade e privacidade

Rotas usarao templates, nunca IDs concretos. `fiscal_document_id`, `request_id`, chave de acesso, CNPJ, `trace_id` e erro livre nao serao labels de metrica.

Metricas agregadas por tenant somente serao criadas quando houver limite conhecido e necessidade operacional. Investigacao individual usara logs e consultas autorizadas, nao labels ilimitadas.

## Dashboards minimos

### Visao do fluxo fiscal

- recebidos, em processamento, autorizados, rejeitados e falhos;
- taxa de sucesso por ambiente e modulo;
- tempo interno e tempo externo separados;
- documentos parados ou com resultado desconhecido.

### API e ingestao

- volume, latencia e erros por rota;
- autenticacao, rate limit e idempotencia;
- falhas de mapping e validacao.

### Workers e fila

- profundidade, idade, throughput e leases;
- retentativas, falhas e reconciliacoes;
- saude e capacidade dos consumidores.

### SEFAZ

- disponibilidade observada por autorizador e servico;
- latencia, timeout, rejeicao e resultado desconhecido;
- diferenca entre falha externa e interna.

### Webhooks

- primeira entrega, sucesso final, retentativa e esgotamento;
- endpoints com falha continua;
- tempo entre evento e notificacao.

### Infraestrutura e seguranca

- banco, armazenamento, backup e migrations;
- certificados a vencer;
- eventos de seguranca e operacoes administrativas relevantes.

## Objetivos de servico

Os objetivos iniciais orientam a implementacao e deverao ser confirmados por teste de carga e operacao de homologacao antes de virarem compromisso contratual.

### Separacao de responsabilidades

- disponibilidade e latencia da API medem somente o que a plataforma controla;
- tempo de emissao separa espera interna, processamento interno e tempo da SEFAZ;
- sucesso de webhook separa criacao e tentativa da plataforma da disponibilidade do destino do cliente;
- rejeicao fiscal valida nao conta como indisponibilidade tecnica;
- manutencoes e exclusoes contratuais deverao ser declaradas, nunca inferidas depois do incidente.

### Metas iniciais do MVP

- disponibilidade mensal da API de ingestao em producao: `99,5%`;
- percentil 95 da aceitacao e persistencia de uma solicitacao valida: ate `500 ms` sob limites homologados, sem aguardar SEFAZ;
- percentil 99 dos eventos internos disponiveis para processamento: ate `60 segundos` quando a plataforma estiver saudavel;
- primeira tentativa de webhook final: ate `60 segundos` apos persistencia do evento, quando o destino puder ser resolvido;
- RPO do banco: ate 15 minutos;
- RPO proximo de zero para XML confirmado;
- RTO geral: ate quatro horas.

As tres ultimas metas preservam as decisoes da Etapa 8. Nenhuma meta promete prazo de autorizacao fiscal enquanto a SEFAZ estiver processando ou indisponivel.

## Alertas

### Regras gerais

Todo alerta devera possuir:

- nome e condicao claras;
- severidade;
- janela e limiar;
- componente e ambiente;
- impacto esperado;
- link para dashboard e runbook;
- responsavel ou escala;
- criterio de encerramento;
- protecao contra duplicacao e tempestade de alertas.

Limiares de taxa serao ajustados apos baseline. Condicoes de integridade e seguranca nao dependerao de grande volume para alertar.

### Alertas obrigatorios

- protocolo ou XML autorizado sem persistencia confirmada;
- documento em `authorization_unknown` alem do limite operacional;
- crescimento ou idade excessiva da fila;
- worker sem progresso;
- aumento anormal de falha interna;
- aumento anormal de rejeicoes por uma mesma categoria;
- falha continua ou latencia anormal da SEFAZ;
- retentativas esgotadas;
- entregas de webhook esgotadas ou endpoint degradado;
- certificado proximo do vencimento, vencido ou inacessivel;
- banco ou armazenamento indisponivel;
- backup, restauracao de teste ou migration com falha;
- tenant com drift ou versao incompatível;
- tentativa de acesso entre tenants;
- falha de descriptografia ou indicio de segredo em log.

Um documento isolado com erro corrigivel pelo cliente nao gerara alerta de plantao. Acumulo, regressao, violacao de integridade ou falha sistemica podera gerar.

## Severidade e atendimento

| Severidade | Criterio | Exemplos |
| --- | --- | --- |
| `P1` | Risco de seguranca, perda ou adulteracao de dados, duplicidade fiscal, ou emissao global parada. | Acesso entre tenants, XML autorizado perdido, retransmissao insegura, banco indisponivel. |
| `P2` | Tenant ou empresa impedido de emitir sem alternativa segura. | Certificado invalido, fila presa para um grupo, autorizador especifico indisponivel sem contingencia habilitada. |
| `P3` | Falha isolada, degradacao com alternativa ou problema de integracao. | Webhook esgotado, rejeicoes por configuracao, latencia elevada. |
| `P4` | Duvida, melhoria ou problema sem impacto operacional imediato. | Ajuste de mensagem ou pedido de relatorio. |

### Objetivos de resposta

Antes da primeira producao, a plataforma devera declarar horario de cobertura, canal e responsaveis. Metas iniciais de reconhecimento durante a cobertura:

- `P1`: ate 15 minutos;
- `P2`: ate uma hora;
- `P3`: ate um dia util;
- `P4`: ate tres dias uteis.

Essas metas sao de reconhecimento e inicio de diagnostico, nao promessa de resolucao. Atendimento fora da cobertura somente sera prometido quando existir plantao definido, testado e compativel com o contrato do cliente.

## Fluxo de incidente

1. Detectar por alerta, cliente ou verificacao operacional.
2. Registrar incidente, horario, severidade e responsavel.
3. Conter o risco sem retransmitir ou alterar estado fiscal de forma insegura.
4. Preservar evidencias, identificadores e historico.
5. Diagnosticar separando plataforma, configuracao, cliente, infraestrutura e SEFAZ.
6. Comunicar internamente e ao grupo conforme impacto e politica da Etapa 8.
7. Recuperar por operacao suportada e auditada.
8. Confirmar integridade de estado, protocolo, XML e notificacoes.
9. Encerrar com causa, impacto, linha do tempo e acoes preventivas.
10. Realizar revisao posterior para `P1` e `P2` relevantes.

Incidentes com dados pessoais ou seguranca seguirao tambem os prazos, papeis e criterios de comunicacao definidos na Etapa 8.

## Runbooks obrigatorios

- SEFAZ indisponivel ou com latencia elevada;
- autorizacao com resultado desconhecido;
- documento ou fila sem progresso;
- worker interrompido antes ou depois da transmissao;
- retentativas esgotadas;
- protocolo ou XML sem persistencia confirmada;
- reprocessamento manual seguro;
- webhook falhando ou endpoint desativado;
- certificado a vencer, vencido, invalido ou comprometido;
- banco ou armazenamento indisponivel;
- restauracao e verificacao de integridade;
- migration falha ou tenant com drift;
- suspeita de acesso entre tenants;
- credencial ou segredo exposto;
- cancelamento ou inutilizacao com resultado desconhecido.

Cada runbook deve conter:

```text
objetivo e sintomas
pre-condicoes e riscos
consultas e dashboards permitidos
passos de diagnostico
acoes seguras
acoes proibidas
criterio de escalacao
criterio de recuperacao
validacao posterior
evidencias e auditoria exigidas
responsavel e ultima revisao
```

## Operacoes administrativas do MVP

O MVP devera permitir, por comando interno ou API administrativa protegida:

- localizar documento por IDs autorizados e referencia externa;
- consultar estado interno, transicoes, tentativas e respostas tecnicas mascaradas;
- solicitar reconciliacao de resultado desconhecido;
- solicitar reprocessamento somente em estado e erro permitidos;
- reenviar evento de webhook;
- testar, desativar e reativar endpoint de webhook;
- suspender integracao, empresa, grupo ou API key conforme permissao;
- consultar metadados e validade do certificado sem expor o PFX;
- verificar versao de migration e drift do tenant;
- consultar saude da fila, worker, banco e armazenamento;
- registrar incidente e vincular recursos afetados.

Regras obrigatorias:

- autenticacao administrativa separada da API key de integracao;
- autorizacao conforme os perfis definidos na Etapa 8;
- justificativa para acao que altere processamento, acesso ou configuracao critica;
- confirmacao explicita para operacoes de maior risco;
- ator, momento, recurso, resultado, motivo e correlacao na auditoria;
- idempotencia quando a repeticao puder produzir novo efeito;
- proibicao de editar status, numero, chave, protocolo ou historico diretamente;
- nenhuma operacao administrativa pode ignorar isolamento de tenant.

O painel administrativo visual fica adiado. A futura interface devera consumir as mesmas operacoes protegidas, sem possuir caminho privilegiado proprio.

## Auditoria e historico operacional

### Historico fiscal

Transicoes de estado, tentativas, respostas relevantes, hashes e referencias de artefato continuam imutaveis e vinculados ao documento.

### Auditoria administrativa e de seguranca

Registra quem executou ou tentou executar uma operacao sensivel, conforme a Etapa 8. Reconciliacao manual, reprocessamento, reenvio de webhook, alteracao de endpoint, suspensao e mudanca de severidade serao eventos auditaveis.

### Logs operacionais

Explicam o comportamento tecnico e podem ser amostrados ou expirar. Nao substituem o historico fiscal nem a auditoria.

Um log nunca sera a unica evidencia de autorizacao, rejeicao, cancelamento, inutilizacao ou acao administrativa critica.

## Estrategia de testes

### Consulta e erros

- status publico corresponde ao estado interno;
- estado de outro tenant retorna resposta indistinguivel de recurso inexistente;
- `authorized` somente aparece com XML e protocolo persistidos;
- codigos de erro permanecem estaveis;
- erros nao expõem stack, SQL, schema ou dado sensivel;
- orientacao de retentativa corresponde a politica real.

### Correlacao e observabilidade

- IDs atravessam API, outbox, worker, SEFAZ, armazenamento e webhook;
- trace interrompido nao impede auditoria por IDs persistentes;
- logs estruturados possuem campos obrigatorios;
- dados proibidos nao aparecem em log, metrica, trace ou alerta;
- labels respeitam limites de cardinalidade;
- dashboards diferenciam falha interna e externa.

### Webhooks

- assinatura valida e invalida;
- corpo alterado depois da assinatura;
- timestamp expirado e tentativa de replay;
- evento duplicado e deduplicacao por `event_id`;
- timeout, `429`, `5xx`, `4xx` e `3xx`;
- backoff, jitter e limite de tentativas;
- reenvio manual preservando evento original;
- rotacao de segredo durante sobreposicao;
- URL privada, loopback, link-local, redirecionamento e DNS rebinding;
- separacao entre homologacao e producao;
- endpoint lento ou com resposta excessiva;
- evento salvo mesmo com publicador interrompido.

### Alertas e runbooks

- cada alerta critico e disparado em ambiente de teste;
- notificacao chega ao canal e responsavel esperados;
- deduplicacao e encerramento funcionam;
- link do alerta abre dashboard e runbook corretos;
- simulacao executa pelo menos resultado desconhecido, fila parada, falha de persistencia e certificado a vencer;
- evidencias e auditoria permanecem depois da recuperacao.

### Recuperacao operacional

- reconciliar sem retransmissao cega;
- reprocessar somente erro permitido;
- recuperar lease expirada;
- restaurar banco em ambiente isolado;
- verificar hash e acesso aos XMLs depois da restauracao;
- retomar webhook sem perder nem criar evento de negocio duplicado;
- recuperar tenant de migration falha sem afetar os demais.

## Criterios para concluir a implementacao da Etapa 10

O planejamento fica concluido com este documento. A implementacao somente podera mover a etapa para `docs/implementado` quando houver:

1. endpoint de consulta com estados e erros publicos validados;
2. correlacao ponta a ponta demonstrada;
3. catalogo de erros publicado e testado;
4. configuracao, assinatura, entrega, retentativa e reenvio de webhook implementados;
5. protecoes de destino externo e segredo de webhook validadas;
6. logs estruturados e mascaramento centralizado;
7. metricas e traces essenciais sem dados sensiveis;
8. dashboards minimos disponiveis;
9. alertas criticos exercitados;
10. runbooks obrigatorios revisados;
11. operacoes administrativas minimas protegidas e auditadas;
12. severidade, cobertura, canal e responsaveis definidos;
13. simulacao de incidente e recuperacao registrada;
14. objetivos de servico revisados com baseline de homologacao;
15. confirmacao de que status autorizado exige XML e protocolo persistidos;
16. testes de isolamento multi-tenant em consulta, operacao e webhook.

## Decisoes adiadas conscientemente

- painel administrativo visual completo;
- portal self-service de suporte;
- webhooks intermediarios de todos os estados;
- ordenacao global de eventos;
- multiplos endpoints por integracao e ambiente;
- regras de transformacao customizada do payload de webhook;
- analytics fiscal e comercial;
- data warehouse de observabilidade;
- SIEM e SOC dedicados;
- operacao ativa multi-regiao;
- SLA individual por autorizador fiscal;
- suporte 24x7 sem equipe e plantao formalizados;
- remediacao automatica de rejeicao fiscal;
- ativacao automatica irrestrita de contingencia.

Esses itens poderao ser antecipados por risco, contrato, volume ou exigencia legal, mas nao bloqueiam o primeiro MVP.

## Decisoes consolidadas da Etapa 10

1. A API de consulta sera a fonte de verdade; webhook sera notificacao pelo menos uma vez.
2. O fluxo usara IDs persistentes de negocio e IDs tecnicos de trace com finalidades separadas.
3. O formato de erro da Etapa 6 sera preservado e ampliado com categoria, retentabilidade e correlacao.
4. O MVP notificara apenas autorizacao, rejeicao, falha e cancelamento.
5. Webhooks serao assinados por HMAC-SHA256, versionados e protegidos contra replay e SSRF.
6. Eventos de webhook nascerao da outbox transacional definida na Etapa 9.
7. Entrega sera pelo menos uma vez, sem garantia de ordenacao global e com deduplicacao por `event_id`.
8. Logs, metricas, traces, historico fiscal e auditoria serao separados.
9. Observabilidade nao armazenara segredo, XML completo ou payload completo.
10. Metricas distinguirao tempo interno, SEFAZ e destino de webhook.
11. Alertas serao acionaveis e vinculados a runbooks.
12. A severidade inicial sera dividida entre `P1`, `P2`, `P3` e `P4`.
13. Operacoes administrativas exigirao permissao, justificativa e auditoria.
14. Nenhum operador alterara diretamente estado fiscal ou historico no banco.
15. O painel visual sera adiado, mas as capacidades operacionais farao parte do MVP.
16. Metas iniciais serao validadas por baseline antes de compromisso contratual.
17. A plataforma nao prometera tempo de autorizacao que dependa da SEFAZ.
18. Runbooks e simulacoes de falha serao criterios de prontidao para producao.

## Resultado esperado

A implementacao tera os meios necessarios para acompanhar uma emissao, distinguir falha interna de dependencia externa, reconciliar resultados desconhecidos, comunicar o cliente, detectar degradacao e executar recuperacao segura sem manipular diretamente o banco.

O proximo planejamento podera transformar todas as decisoes das Etapas 1 a 10 em um backlog tecnico do MVP, incluindo como entregas obrigatorias a observabilidade, os webhooks, os runbooks e as operacoes administrativas minimas definidas aqui.
