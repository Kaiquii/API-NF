# Pendencias de planejamento

Este documento e o roteiro de planejamento da API de emissao fiscal. Vamos seguir as etapas na ordem abaixo e registrar cada decisao no documento especifico correspondente.

Uma etapa so deve ser considerada concluida quando suas decisoes estiverem documentadas e nao deixarem duvidas para a implementacao.

## Como vamos usar este roteiro

- `[ ]` pendente;
- `[~]` em planejamento;
- `[x]` planejado e pronto para implementacao.

Quando uma etapa for concluida, seu documento deve sair desta pasta e ir para `docs/planejado-falta-implementar`. Depois de implementada e validada, vai para `docs/implementado`.

## Sequencia de planejamento

### Etapa 1 - Fundacao de flexibilidade e conformidade fiscal

Status: `[x]`

Definir como a plataforma pode ser flexivel para usuarios e integracoes sem permitir desvios das exigencias legais e fiscais.

As decisoes desta etapa estao documentadas em `docs/planejado-falta-implementar/FUNDACAO_FLEXIBILIDADE_E_CONFORMIDADE_FISCAL.md`.

Resultado: plataforma configuravel na entrada, com nucleo fiscal interno, validacao obrigatoria, regras versionadas e modulos fiscais independentes.

### Etapa 2 - Consolidacao de dominio e nomenclatura

Status: `[x]`

Padronizar os conceitos tecnicos da plataforma e remover ambiguidades entre grupo, cliente, empresa, integracao, nota e documento fiscal.

As decisoes desta etapa estao documentadas em `docs/planejado-falta-implementar/DECISOES_DE_DOMINIO_E_NOMENCLATURA.md`.

Resultado: `group` identifica o tenant, `company` identifica o emitente, `integration` identifica a entrada configurada e `fiscal document` identifica o recurso generico de emissao.

### Etapa 3 - Modelo interno do documento fiscal

Status: `[x]`

Definir o contrato canonico do documento fiscal dentro da plataforma. Ele deve conter, no minimo, emitente, destinatario, itens, impostos, frete, pagamentos, totais, informacoes adicionais e referencias fiscais.

Todo payload recebido, seja padrao ou customizado, deve ser convertido para esse modelo antes do processamento fiscal.

As decisoes desta etapa estao documentadas em `docs/planejado-falta-implementar/MODELO_INTERNO_DE_DOCUMENTOS_FISCAIS.md`.

Resultado: envelope comum, contratos por modulo, referencia `nfe/v1`, regras de normalizacao, rastreabilidade e versionamento.

### Etapa 4 - Regras de preenchimento fiscal

Status: `[~]`

Definir a origem de cada informacao obrigatoria da nota.

Exemplos:

- dados comerciais recebidos no payload;
- NCM, unidade e origem obtidos no cadastro de produtos;
- CFOP, CST, CSOSN e aliquotas definidos por regras fiscais;
- serie e numeracao controladas pela empresa emitente.

Tambem deve ficar claro quais campos sao obrigatorios para o cliente e quais podem ser completados pela plataforma.

A direcao desta etapa esta documentada em `docs/pendente-planejamento/REGRAS_DE_PREENCHIMENTO_FISCAL.md`. A etapa permanece em planejamento ate a definicao da empresa real do MVP e a conclusao da matriz de campos do contrato `nfe/v1`.

Resultado esperado: matriz de responsabilidade de cada campo fiscal, incluindo origem, regra de preenchimento e validacao.

### Etapa 5 - Numeracao, idempotencia e maquina de estados

Status: `[ ]`

Definir como a numeracao sera reservada por empresa, serie e modelo, evitando colisao em chamadas simultaneas.

Definir tambem a estrategia de idempotencia: uma repeticao da mesma requisicao pelo ERP, por exemplo apos um timeout, nao pode resultar em duas notas emitidas.

Erros temporarios de comunicacao com a SEFAZ podem ser retentados. Rejeicoes fiscais precisam retornar ao cliente para correcao, sem nova tentativa automatica.

Resultado esperado: diagrama de estados, regras de transicao, politica de retentativas e estrategia de reserva de numeracao.

### Etapa 6 - Contratos da API e mapeamento de integracoes

Status: `[ ]`

Definir os limites do `mapping_config`: ele deve servir para mapear e normalizar formatos de entrada, sem se tornar uma camada de regras de negocio arbitrarias por cliente.

Casos que nao puderem ser resolvidos por configuracao devem ter um processo definido para adaptadores especificos, versionados e testados.

Esta etapa tambem deve detalhar endpoints externos, formato de respostas, erros, versionamento de API, idempotency key e uso de `company_id` e `integration_id`.

Resultado esperado: contrato da API padrao e especificacao da integracao customizada.

### Etapa 7 - Integracao com a SEFAZ

Status: `[ ]`

Planejar a separacao entre geracao de XML, assinatura com certificado A1, comunicacao com servicos da SEFAZ, consulta de recibo, armazenamento do XML autorizado e contingencia.

Esse fluxo deve ser executado por workers assincronos, e nao durante a requisicao HTTP do cliente.

Resultado esperado: fluxo tecnico de emissao, contratos entre API e worker, tratamento de retorno e erros da SEFAZ.

### Etapa 8 - Seguranca e dados sensiveis

Status: `[ ]`

Definir como certificados digitais e respectivas senhas serao criptografados, acessados e auditados. Tambem inclui gestao de segredos, retencao de XMLs e payloads, mascaramento de logs e permissoes administrativas.

Resultado esperado: politica de seguranca para certificados, dados fiscais, logs, auditoria e acesso administrativo.

### Etapa 9 - Banco de dados e migrations multi-tenant

Status: `[ ]`

Definir o modelo fisico das tabelas do schema `platform` e dos schemas dos grupos, suas chaves, indices, restricoes e estrategia de migracao.

As migrations precisam ser aplicadas no schema central e em todos os schemas de grupos com versao, execucao segura, acompanhamento de falhas e possibilidade de recuperacao. Novos grupos devem nascer na versao mais recente.

Resultado esperado: modelo de dados fisico e processo confiavel de provisionamento e atualizacao de schemas.

### Etapa 10 - Operacao e suporte

Status: `[ ]`

Definir os recursos para acompanhamento da emissao:

- consulta de status;
- webhooks;
- correlacao entre requisicao do ERP e nota fiscal;
- mensagens de erro compreensiveis;
- auditoria;
- metricas, alertas e logs operacionais;
- painel administrativo futuro.

Resultado esperado: definicao de observabilidade, atendimento de falhas, webhooks e recursos administrativos necessarios para operar o produto.

### Etapa 11 - Plano de implementacao do MVP

Status: `[ ]`

Transformar as decisoes anteriores em fases tecnicas de implementacao, com dependencias, entregas e criterios de validacao.

Resultado esperado: backlog tecnico do MVP, organizado em ordem de execucao.

## Planejamentos ja definidos

Os documentos abaixo ja possuem uma direcao arquitetural e estao em `docs/planejado-falta-implementar`:

- planejamento geral da API fiscal;
- multi-tenancy por grupos e schemas;
- autenticacao e gerenciamento de API Keys.
