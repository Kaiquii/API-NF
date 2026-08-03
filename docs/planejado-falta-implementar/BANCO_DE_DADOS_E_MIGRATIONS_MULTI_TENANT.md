# Banco de dados e migrations multi-tenant

> Status: planejamento concluído, aguardando implementação.

## Objetivo

Definir o modelo físico inicial do PostgreSQL e o processo de provisionamento e evolução dos schemas da plataforma e dos grupos.

Esta etapa transforma as decisões de domínio, segurança, idempotência, numeração e integração fiscal em estruturas persistentes, invariantes e procedimentos verificáveis. A implementação poderá acrescentar colunas auxiliares, desde que não altere as fronteiras e garantias definidas aqui sem uma nova decisão arquitetural registrada.

## Princípios

- O schema `platform` contém apenas identidade, autenticação, administração, auditoria central e controle da evolução dos tenants.
- Cada grupo possui exatamente um schema `tenant_<uuid_sem_hifens>`.
- Dados comerciais, fiscais, certificados e artefatos de um grupo ficam somente no schema desse grupo.
- XMLs ficam em armazenamento de objetos privado; o PostgreSQL guarda metadados, referência e hash.
- O banco é a autoridade final para unicidade, idempotência, numeração e concorrência.
- Toda tabela possui chave primária, datas em UTC e invariantes explícitas.
- Registros fiscais e históricos imutáveis não são atualizados nem excluídos pela aplicação.
- Migrations são versionadas, repetíveis de forma segura e aplicadas com exclusão mútua.
- Um tenant com falha de migration não impede a evolução dos demais e não pode operar com versão incompatível.
- Nenhum identificador de schema recebido externamente será considerado confiável.

## Base tecnológica

### PostgreSQL

O MVP usará PostgreSQL 18, sempre na versão minor mais recente disponível no ambiente. Produção, desenvolvimento, CI e homologação usarão a mesma versão major; suporte a outra versão exigirá decisão e testes próprios.

Antes da primeira produção, o provedor escolhido deverá confirmar suporte a:

- backups com recuperação ponto no tempo;
- conexões TLS;
- métricas e logs administrativos;
- armazenamento criptografado;
- restauração em ambiente isolado;
- extensões aprovadas pelo projeto.

### Acesso pelo Go

- `pgx/v5` e `pgxpool` serão usados para conexão e pool.
- Consultas serão SQL explícito e parametrizado.
- Um gerador como `sqlc` poderá ser adotado na implementação, sem alterar o modelo físico.
- ORM não será requisito do MVP.
- Toda operação em tenant será executada dentro de transação explícita.

### Migrations

O padrão será SQL-first com Goose, usando migrations embarcadas no binário administrativo. Haverá dois conjuntos independentes:

```text
migrations/
├── platform/
└── tenant/
```

O projeto possuirá um comando Go próprio para migrations e provisionamento. O binário da API não executará DDL automaticamente durante sua inicialização.

O migrador deverá usar lock de sessão do PostgreSQL para impedir duas execuções globais concorrentes. Migrations fora de transação serão excepcionais, precisarão de justificativa no arquivo e terão procedimento específico de recuperação.

## Convenções físicas

### Nomes

- schemas, tabelas, colunas, índices e constraints usarão `snake_case` em inglês;
- tabelas usarão plural;
- chaves primárias: `pk_<table>`;
- chaves estrangeiras: `fk_<table>__<referenced_table>`;
- unicidades: `uq_<table>__<columns>`;
- checks: `ck_<table>__<rule>`;
- índices: `idx_<table>__<columns_or_purpose>`;
- nenhum nome comercial será usado em identificador físico.

### Identificadores

- identificadores de domínio usarão `uuid`;
- UUIDs serão versão 7 e gerados pela aplicação;
- o banco não aceitará UUID vazio;
- identificadores fiscais externos, como chave de acesso, recibo e protocolo, serão `text`, pois não participam de cálculos;
- números sequenciais internos de tentativas e versões usarão `bigint` ou `integer`, conforme o escopo.

### Datas e tempo

- instantes usarão `timestamptz` e serão persistidos em UTC;
- datas fiscais sem horário usarão `date`;
- durações usarão milissegundos em `bigint` quando precisarem ser armazenadas;
- todas as tabelas mutáveis terão `created_at` e `updated_at`;
- tabelas imutáveis terão `created_at` ou `occurred_at`, sem `updated_at`.

### Valores numéricos

- valores monetários consolidados: `numeric(20,2)`;
- valores unitários e bases intermediárias: `numeric(20,10)`;
- quantidades: `numeric(20,6)`;
- alíquotas: `numeric(12,8)`;
- contadores e números fiscais: `bigint`;
- `real`, `double precision` e tipos de ponto flutuante não serão usados em cálculos fiscais.

O snapshot fiscal preservará também os valores normalizados e as regras de arredondamento usadas pelo módulo.

### Texto e documentos

- CPF, CNPJ, CEP, telefone, NCM, CFOP, CST, CSOSN e códigos fiscais serão `text` normalizado;
- CNPJ terá 14 dígitos e CPF terá 11 dígitos quando presentes;
- validação de dígitos verificadores continuará no domínio, enquanto formato e tamanho terão `CHECK` no banco;
- campos livres terão limite explícito, nunca `text` ilimitado por omissão sem justificativa.

### Status

Status serão armazenados como `text` com `CHECK`, e não como `ENUM` do PostgreSQL. Essa escolha facilita evolução compatível e rollback lógico. A aplicação continuará usando tipos fechados no código.

### Uso de JSONB

`jsonb` será permitido para:

- payload original e representações normalizadas versionadas;
- snapshots fiscais imutáveis;
- configurações declarativas de mapeamento e regras;
- erros estruturados e metadados variáveis;
- respostas técnicas necessárias à auditoria.

`jsonb` não substituirá colunas usadas em identidade, relacionamento, autorização, estado, concorrência, retenção, numeração ou consultas operacionais frequentes.

Todo JSON persistido terá versão de contrato associada e limite de tamanho aplicado antes da gravação.

## Organização do banco

```text
database api_nf
│
├── platform
│   ├── groups
│   ├── api_keys
│   ├── admin_identities
│   ├── audit_events
│   └── tenant_migration_runs
│
├── tenant_<group_uuid>
│   ├── companies e configurações
│   ├── integrations e versões
│   ├── certificates
│   ├── cadastros auxiliares e regras fiscais
│   ├── solicitações, documentos e itens
│   ├── snapshots, tentativas e transições
│   ├── eventos fiscais e inutilizações
│   ├── artefatos fiscais
│   └── outbox e concessões de processamento
│
└── outros schemas de tenant com a mesma versão estrutural
```

Não haverá tabelas de negócio no schema `public`. O privilégio `CREATE` será revogado de `PUBLIC`, e os papéis de runtime não usarão `public` no `search_path`.

## Modelo físico do schema `platform`

### `groups`

| Coluna | Tipo | Regra |
| --- | --- | --- |
| `id` | `uuid` | PK, UUID v7. |
| `name` | `varchar(200)` | Nome administrativo; não forma o schema. |
| `schema_name` | `varchar(80)` | Único, gerado internamente, formato controlado. |
| `status` | `varchar(32)` | `provisioning`, `active`, `migration_pending`, `migrating`, `migration_failed`, `suspended`, `cancelled` ou `error`. |
| `current_schema_version` | `bigint` | Versão aplicada com sucesso. |
| `target_schema_version` | `bigint` | Versão esperada pelo processo em andamento. |
| `minimum_compatible_version` | `bigint` | Menor versão aceita pelo deploy atual. |
| `migration_error_code` | `varchar(100)` | Código sanitizado da última falha. |
| `migration_error_at` | `timestamptz` | Data da última falha. |
| `activated_at` | `timestamptz` | Primeiro provisionamento concluído. |
| `suspended_at` | `timestamptz` | Data de suspensão. |
| `cancelled_at` | `timestamptz` | Data de cancelamento lógico. |
| `created_at` | `timestamptz` | Obrigatório. |
| `updated_at` | `timestamptz` | Obrigatório. |

Invariantes:

- `schema_name` deve corresponder a `^tenant_[0-9a-f]{32}$`;
- `schema_name` é único e imutável;
- grupo diferente de `active` não consome endpoints externos;
- grupo com `current_schema_version < minimum_compatible_version` não opera;
- suspensão e cancelamento não removem o schema.

### `api_keys`

| Coluna | Tipo | Regra |
| --- | --- | --- |
| `id` | `uuid` | PK; parte pública da credencial. |
| `group_id` | `uuid` | FK para `groups`. |
| `environment` | `varchar(16)` | `test` ou `production`. |
| `prefix` | `varchar(80)` | Único, seguro para suporte. |
| `secret_digest` | `bytea` | Hash ou HMAC; segredo nunca é persistido. |
| `digest_algorithm` | `varchar(32)` | Versão do mecanismo de verificação. |
| `status` | `varchar(16)` | `active`, `revoked`, `expired` ou `deleted`. |
| `replaces_key_id` | `uuid` | FK opcional para a chave anterior. |
| `rotation_expires_at` | `timestamptz` | Obrigatório para a chave anterior durante rotação. |
| `last_used_at` | `timestamptz` | Atualização assíncrona permitida. |
| `revoked_at` | `timestamptz` | Obrigatório quando revogada. |
| `revoked_by_admin_id` | `uuid` | Ator administrativo, quando aplicável. |
| `revoke_reason` | `varchar(500)` | Motivo administrativo. |
| `deleted_at` | `timestamptz` | Exclusão lógica. |
| `deleted_by_admin_id` | `uuid` | Ator da exclusão lógica. |
| `created_at` | `timestamptz` | Obrigatório. |

Invariantes:

- prefixo único localiza uma única credencial;
- uma chave funciona somente no ambiente registrado;
- `deleted` exige revogação prévia;
- rotação é protegida com lock da linha do grupo e transação;
- fora de rotação existe no máximo uma chave ativa por grupo e ambiente;
- durante rotação existem no máximo duas, com expiração obrigatória da anterior;
- a expiração efetiva também é verificada em toda autenticação, sem depender apenas do job de limpeza.

Restrições parciais e transações implementarão essas regras. Como a condição de duas chaves depende da relação de substituição e prazo, ela não será confiada apenas a um índice isolado.

### `admin_identities`

Esta tabela não implementa login. Ela mantém a referência estável do ator administrativo usada por auditoria e FKs até a escolha do provedor de identidade.

```text
id uuid PK
external_subject varchar(255) UNIQUE
display_name varchar(200)
status varchar(16)
created_at timestamptz
updated_at timestamptz
```

Papéis e permissões detalhados poderão vir do provedor de identidade, mas a decisão aplicada e o ator serão preservados nos eventos de auditoria.

### `audit_events`

Tabela append-only para eventos administrativos e de segurança que atravessam tenants.

```text
id uuid PK
group_id uuid NULL
company_id uuid NULL
actor_type varchar(32)
actor_id varchar(255)
action varchar(120)
resource_type varchar(80)
resource_id varchar(255) NULL
result varchar(32)
reason_code varchar(100) NULL
correlation_id uuid NULL
source_ip inet NULL
metadata jsonb
occurred_at timestamptz
```

Não haverá FK de `company_id` para schemas de tenants. O identificador é preservado como referência auditável. A tabela não armazenará segredos, payloads, XMLs ou dados pessoais integrais.

`UPDATE` e `DELETE` serão negados aos papéis de runtime. Correções geram um novo evento.

### `tenant_migration_runs`

Registra cada tentativa de migration por tenant.

```text
id uuid PK
group_id uuid FK
from_version bigint
to_version bigint
status varchar(24)
runner_version varchar(80)
started_at timestamptz
finished_at timestamptz NULL
duration_ms bigint NULL
error_code varchar(100) NULL
error_detail_redacted varchar(1000) NULL
created_at timestamptz
```

O histórico é imutável. A versão registrada dentro do schema do tenant é a autoridade da estrutura aplicada; `groups.current_schema_version` é a projeção central usada para roteamento e acompanhamento.

## Modelo físico de cada tenant

Todas as FKs abaixo são internas ao mesmo schema. Não haverá FK atravessando schemas de tenants ou ligando tabelas fiscais ao schema `platform`.

### Empresas e configurações

#### `companies`

```text
id uuid PK
cnpj varchar(14) NOT NULL
legal_name varchar(200) NOT NULL
trade_name varchar(200) NULL
state_registration varchar(30) NULL
municipal_registration varchar(30) NULL
state_code char(2) NOT NULL
tax_regime varchar(32) NOT NULL
status varchar(16) NOT NULL
created_at timestamptz
updated_at timestamptz
disabled_at timestamptz NULL
```

- CNPJ será único entre empresas não removidas logicamente no tenant;
- `status`: `active`, `inactive` ou `blocked`;
- empresa com movimentação fiscal não será excluída fisicamente.

#### `company_fiscal_config_versions`

Versões imutáveis das configurações que influenciam preenchimento, cálculo, numeração e transmissão.

```text
id uuid PK
company_id uuid FK
version integer
environment varchar(16)
document_type varchar(20)
config jsonb
status varchar(16)
valid_from timestamptz
valid_until timestamptz NULL
created_by varchar(255)
created_at timestamptz
```

Somente uma versão poderá estar ativa por empresa, ambiente e tipo de documento. Versões usadas por documentos não serão alteradas.

### Integrações e mapeamentos

#### `integrations`

```text
id uuid PK
name varchar(200)
input_type varchar(16)
environment varchar(16)
adapter_code varchar(100) NULL
status varchar(16)
created_at timestamptz
updated_at timestamptz
retired_at timestamptz NULL
```

`input_type` será `standard` ou `custom`. Integrações referenciadas por documentos são aposentadas, não removidas.

#### `integration_mapping_versions`

```text
id uuid PK
integration_id uuid FK
version integer
target_contract varchar(40)
mapping_config jsonb
mapping_hash char(64)
status varchar(16)
activated_at timestamptz NULL
retired_at timestamptz NULL
created_by varchar(255)
created_at timestamptz
```

- status: `draft`, `testing`, `active` ou `retired`;
- versão única por integração;
- somente uma versão ativa por integração, ambiente e contrato de destino;
- versões ativas ou usadas são imutáveis;
- casos de teste serão armazenados em `integration_mapping_test_cases`.

#### `integration_mapping_test_cases`

```text
id uuid PK
mapping_version_id uuid FK
name varchar(200)
input_payload jsonb
expected_output jsonb NULL
expected_errors jsonb NULL
created_at timestamptz
```

Dados de teste produtivos e segredos são proibidos.

### Certificados

#### `certificates`

```text
id uuid PK
company_id uuid FK
environment varchar(16)
version integer
status varchar(20)
subject_cnpj varchar(14)
serial_number varchar(200)
issuer varchar(500)
valid_from timestamptz
valid_until timestamptz
algorithm varchar(40)
key_reference varchar(500)
encrypted_data_key bytea
nonce bytea
ciphertext bytea
auth_tag bytea
aad_version integer
ciphertext_hash char(64)
replaces_certificate_id uuid NULL FK
blocked_at timestamptz NULL
retired_at timestamptz NULL
destroy_after timestamptz NULL
destroyed_at timestamptz NULL
created_by varchar(255)
created_at timestamptz
```

- uma versão corresponde a um pacote criptográfico imutável;
- somente um certificado pode estar ativo por empresa, ambiente e finalidade no MVP;
- o contexto autenticado da criptografia inclui tenant, empresa, certificado e ambiente;
- senha, PFX e chave privada em texto puro nunca possuem coluna;
- eliminação criptográfica remove `ciphertext` e `encrypted_data_key`, preservando metadados e auditoria.

### Cadastros e regras fiscais

#### `products`

```text
id uuid PK
company_id uuid FK
sku varchar(120)
description varchar(500)
ncm varchar(8)
default_cfop varchar(4) NULL
commercial_unit varchar(10)
origin_code varchar(2)
cst varchar(3) NULL
csosn varchar(4) NULL
status varchar(16)
created_at timestamptz
updated_at timestamptz
```

SKU será único por empresa entre registros ativos.

#### `recipients`

O nome canônico será `recipients`, em conformidade com o domínio fiscal; o nome anterior `customers` fica descartado.

```text
id uuid PK
tax_id varchar(14)
name varchar(200)
state_registration varchar(30) NULL
state_registration_indicator varchar(4) NULL
email varchar(320) NULL
phone varchar(30) NULL
address jsonb
status varchar(16)
created_at timestamptz
updated_at timestamptz
```

O documento emitido nunca dependerá da versão atual desse cadastro: os dados usados ficam congelados no snapshot fiscal.

#### `tax_rule_versions`

```text
id uuid PK
company_id uuid NULL FK
rule_code varchar(100)
version integer
priority integer
scope jsonb
conditions jsonb
result jsonb
status varchar(16)
valid_from timestamptz
valid_until timestamptz NULL
created_by varchar(255)
created_at timestamptz
```

Regras publicadas e usadas serão imutáveis. A resolução armazenará os IDs e versões efetivamente aplicados.

### Solicitações e idempotência

#### `fiscal_document_requests`

```text
id uuid PK
company_id uuid FK
integration_id uuid NULL FK
mapping_version_id uuid NULL FK
document_type varchar(20)
document_version varchar(20)
operation varchar(40)
environment varchar(16)
idempotency_key varchar(255)
request_fingerprint char(64)
external_reference varchar(255) NULL
raw_payload jsonb
mapped_payload jsonb NULL
normalized_payload jsonb
validation_errors jsonb NULL
status varchar(24)
finalized_at timestamptz NULL
raw_payload_expires_at timestamptz NULL
legal_hold boolean
created_at timestamptz
updated_at timestamptz
```

A unicidade será:

```text
company_id
+ integration_context
+ document_type
+ operation
+ idempotency_key
```

Como o grupo já é delimitado pelo schema, não existe `group_id` na tabela. `integration_context` será uma coluna derivada e não nula: usa o `integration_id` ou um UUID reservado para o fluxo padrão. Isso evita semântica ambígua de `NULL` em índice único.

Comportamento transacional:

1. inserir a solicitação com a chave e fingerprint;
2. em conflito, bloquear e carregar o registro existente;
3. mesma fingerprint retorna o documento existente;
4. fingerprint diferente retorna conflito `409`;
5. somente a transação vencedora cria documento, reserva número e publica trabalho.

`raw_payload` poderá ser apagado ou anonimizado após a retenção de 180 dias. Fingerprint, normalized payload necessário à auditoria e vínculo com o documento permanecem pelo prazo fiscal aplicável.

### Documento e conteúdo fiscal

#### `fiscal_documents`

```text
id uuid PK
request_id uuid UNIQUE FK
company_id uuid FK
integration_id uuid NULL FK
document_type varchar(20)
document_version varchar(20)
environment varchar(16)
operation varchar(40)
external_reference varchar(255) NULL
public_status varchar(20)
internal_state varchar(40)
model varchar(10) NULL
series integer NULL
number bigint NULL
access_key varchar(60) NULL
receipt_number varchar(100) NULL
protocol_number varchar(100) NULL
authorized_at timestamptz NULL
rejected_at timestamptz NULL
cancelled_at timestamptz NULL
total_amount numeric(20,2)
current_snapshot_id uuid NULL
active_lease_id uuid NULL
lock_version bigint
created_at timestamptz
updated_at timestamptz
```

Invariantes:

- `request_id` cria relação um-para-um com a intenção original;
- a combinação empresa, ambiente, modelo, série e número é única quando o número existir;
- chave de acesso é única quando existir;
- estados e transições seguem a máquina definida na Etapa 5;
- `authorized` exige protocolo, data e artefato autorizado persistido;
- `cancelled` exige evento de cancelamento autorizado;
- identidade fiscal reservada nunca é removida nem reutilizada;
- atualizações concorrentes usam estado esperado e `lock_version`.

#### `fiscal_document_snapshots`

```text
id uuid PK
document_id uuid FK
snapshot_version integer
contract_version varchar(40)
module_version varchar(80)
mapping_version_id uuid NULL
company_config_version_id uuid
schema_package_version varchar(80)
fiscal_rule_manifest_version varchar(80)
snapshot jsonb
snapshot_hash char(64)
created_at timestamptz
```

Snapshots são imutáveis e únicos por documento e versão. O snapshot usado na primeira transmissão não será substituído.

#### Tabelas relacionais do documento

O núcleo manterá colunas operacionais e relacionamentos em tabelas, preservando o snapshot completo como fonte auditável:

- `fiscal_document_parties`: emitente, destinatário e demais participantes congelados;
- `fiscal_document_items`: itens, códigos fiscais, quantidades e valores;
- `fiscal_document_item_taxes`: impostos calculados por item e tributo;
- `fiscal_document_payments`: formas e valores de pagamento;
- `fiscal_document_references`: documentos fiscais referenciados;
- `fiscal_document_totals`: totais calculados por categoria.

Todas possuem `document_id`; itens possuem `line_number` único por documento. Valores críticos usam os tipos numéricos definidos nesta etapa. Essas tabelas não consultam cadastros atuais para reinterpretar documento já processado.

### Numeração

#### `fiscal_number_sequences`

```text
id uuid PK
company_id uuid FK
environment varchar(16)
document_type varchar(20)
model varchar(10)
series integer
next_number bigint
last_reserved_number bigint NULL
lock_version bigint
created_at timestamptz
updated_at timestamptz
```

Constraint única:

```text
company_id + environment + document_type + model + series
```

Reserva recomendada:

```sql
UPDATE fiscal_number_sequences
SET last_reserved_number = next_number,
    next_number = next_number + 1,
    lock_version = lock_version + 1,
    updated_at = now()
WHERE id = $1
RETURNING last_reserved_number;
```

Essa atualização ocorre na mesma transação que vincula o número ao documento e registra a transição para `number_reserved`. O contador nunca diminui. Ajustes manuais e saltos exigem operação administrativa auditada e não alteram números já reservados.

### Tentativas, concessões e estados

#### `processing_leases`

```text
id uuid PK
document_id uuid FK
worker_id varchar(200)
lease_token_hash char(64)
acquired_at timestamptz
expires_at timestamptz
released_at timestamptz NULL
status varchar(16)
created_at timestamptz
```

Somente uma concessão ativa pode existir por documento. Um worker assume lease expirada somente depois de avaliar se houve transmissão e se reconciliação é necessária.

#### `fiscal_document_attempts`

```text
id uuid PK
document_id uuid FK
lease_id uuid NULL FK
attempt_type varchar(32)
attempt_number integer
initial_state varchar(40)
final_state varchar(40) NULL
worker_id varchar(200)
module_version varchar(80)
layout_version varchar(80)
service_catalog_version varchar(80) NULL
authority_code varchar(80) NULL
service_code varchar(80) NULL
endpoint_code varchar(100) NULL
xml_artifact_id uuid NULL
xml_hash char(64) NULL
receipt_number varchar(100) NULL
protocol_number varchar(100) NULL
official_code varchar(40) NULL
official_message varchar(1000) NULL
error_category varchar(80) NULL
next_attempt_at timestamptz NULL
started_at timestamptz
transmitted_at timestamptz NULL
finished_at timestamptz NULL
created_at timestamptz
```

Tentativas são imutáveis depois de finalizadas e possuem sequência única por documento e tipo.

#### `fiscal_document_state_transitions`

```text
id uuid PK
document_id uuid FK
previous_state varchar(40) NULL
new_state varchar(40)
reason_code varchar(100) NULL
actor_type varchar(32)
actor_id varchar(255) NULL
attempt_id uuid NULL FK
metadata jsonb
occurred_at timestamptz
```

Tabela append-only. A atualização de `fiscal_documents.internal_state` e a inserção da transição acontecem na mesma transação.

### Artefatos fiscais

#### `fiscal_artifacts`

```text
id uuid PK
document_id uuid FK
attempt_id uuid NULL FK
event_id uuid NULL FK
artifact_type varchar(40)
storage_provider varchar(40)
storage_bucket varchar(200)
storage_key varchar(1000)
content_type varchar(100)
content_length bigint
sha256 char(64)
encryption_key_reference varchar(500) NULL
status varchar(20)
retention_until date
legal_hold boolean
created_at timestamptz
verified_at timestamptz NULL
destroyed_at timestamptz NULL
```

- `storage_key` é referência interna, nunca URL pública permanente;
- XML autorizado e protocolo são imutáveis;
- um documento só muda para `authorized` na mesma unidade lógica em que artefato e protocolo ficam persistidos com segurança;
- acesso e download geram auditoria;
- hash SHA-256 será verificado na leitura e em rotinas de integridade.

### Eventos fiscais e inutilização

Serão tabelas próprias, sem reabrir ou sobrescrever a emissão:

- `fiscal_events`;
- `fiscal_event_attempts`;
- `fiscal_number_void_requests`.

Cada operação possui idempotência, estado, tentativas, artefatos, protocolo e histórico próprios. Cancelamento altera o status público do documento somente após autorização do evento.

### Processamento assíncrono

#### `outbox_events`

```text
id uuid PK
aggregate_type varchar(40)
aggregate_id uuid
event_type varchar(80)
payload jsonb
status varchar(20)
available_at timestamptz
attempts integer
lease_owner varchar(200) NULL
lease_expires_at timestamptz NULL
published_at timestamptz NULL
last_error_code varchar(100) NULL
created_at timestamptz
```

O payload conterá somente identificadores e contexto técnico mínimo; PFX, senha, API key, XML e payload fiscal completo são proibidos.

Documento, estado inicial e evento de outbox são gravados na mesma transação. A implementação poderá publicar o evento em um broker externo ou processar a tabela diretamente no MVP. Consumidores serão idempotentes.

Aquisição de trabalho usará `FOR UPDATE SKIP LOCKED`, lease com expiração e limite de lote. A escolha do broker e a operação completa da fila serão finalizadas nas Etapas 10 e 11 sem modificar a garantia transacional desta tabela.

## Índices obrigatórios

Além dos índices criados por PKs e unicidades, o primeiro conjunto terá:

### `platform`

- `groups(status, current_schema_version)`;
- `api_keys(prefix)` único;
- `api_keys(group_id, environment, status)`;
- `api_keys(rotation_expires_at)` para chaves expirando;
- `audit_events(group_id, occurred_at desc)`;
- `audit_events(correlation_id)`;
- `tenant_migration_runs(status, started_at)`.

### Tenant

- `companies(cnpj)` único conforme regra de status;
- `integrations(status, environment)`;
- `fiscal_document_requests(company_id, external_reference)`;
- escopo único de idempotência;
- `fiscal_documents(company_id, created_at desc)`;
- `fiscal_documents(public_status, updated_at)`;
- identidade fiscal única;
- `fiscal_documents(access_key)` único quando presente;
- `fiscal_document_attempts(document_id, attempt_number)`;
- `fiscal_document_attempts(next_attempt_at)` quando pendente;
- `fiscal_document_state_transitions(document_id, occurred_at)`;
- `processing_leases(expires_at)` quando ativa;
- `outbox_events(status, available_at)`;
- `fiscal_artifacts(document_id, artifact_type)`.

Índices GIN em JSONB não serão criados preventivamente. Precisam de consulta real, volume e justificativa. Índices parciais serão usados para filas, leases, rotações e registros ativos.

## Seleção segura do tenant

Fluxo obrigatório:

1. validar API key em `platform.api_keys`;
2. carregar grupo e `schema_name` de `platform.groups`;
3. validar formato do schema novamente;
4. confirmar status e compatibilidade estrutural;
5. iniciar transação;
6. executar `SET LOCAL search_path` com identificador produzido por função segura de quoting;
7. confirmar `current_schema()` em testes e operações críticas;
8. executar apenas consultas do tenant;
9. concluir ou desfazer a transação.

O `search_path` da sessão permanecerá vazio ou restrito a schemas não graváveis. `SET search_path` de sessão é proibido no pool. Consultas ao schema central usarão nomes qualificados, como `platform.groups`.

O schema fornece a fronteira principal do MVP. RLS não será adicionada às tabelas de tenant porque não existe mistura de linhas de grupos no mesmo schema. O isolamento depende de privilégios, seleção transacional correta, validação de pertencimento e testes negativos.

## Papéis e permissões do PostgreSQL

Papéis mínimos:

| Papel | Permissões |
| --- | --- |
| `api_nf_owner` | Dono sem login dos objetos. Não usado pela aplicação. |
| `api_nf_migrator` | Executa migrations; DDL sem acesso externo. |
| `api_nf_provisioner` | Cria schemas pelo comando administrativo; não atende requisições. |
| `api_nf_api` | Valida credenciais e opera dados necessários da API; sem DDL. |
| `api_nf_worker` | Processa documentos, tentativas e artefatos; sem DDL e sem gestão de API keys. |
| `api_nf_admin` | Operações administrativas aprovadas; sem propriedade dos schemas. |
| `api_nf_auditor` | Leitura mascarada e auditoria; sem segredos e sem escrita comum. |

Regras:

- runtime não será owner de tabelas ou schemas;
- API e worker não terão `CREATE`, `ALTER`, `DROP` ou `TRUNCATE`;
- criação de schema é exclusiva do provisionador;
- migrations usam credencial separada e auditada;
- `PUBLIC` não recebe `CREATE` nem acesso implícito aos schemas;
- privilégios serão concedidos explicitamente para tabelas e sequências novas;
- ambientes terão bancos, papéis, senhas e chaves diferentes.

## Provisionamento de um grupo

O processo será idempotente e terá atomicidade visível para o negócio:

```text
adquirir lock do group_id
  -> criar groups como provisioning
  -> derivar e validar schema_name
  -> criar schema com owner controlado
  -> aplicar migrations tenant até a versão atual
  -> validar catálogo e versão
  -> conceder privilégios de runtime
  -> atualizar current_schema_version
  -> marcar active
  -> registrar auditoria
```

Enquanto não estiver `active`, nenhuma API key do grupo funcionará.

DDL transacional será usado sempre que possível. Como uma sequência de migrations pode possuir unidades transacionais independentes, a garantia principal será: um grupo parcialmente provisionado nunca fica ativo. Em falha antes da ativação, o comando registra o erro e pode remover com segurança apenas o schema novo e vazio, depois de validar que ele pertence ao mesmo grupo e nunca foi ativado.

Retentativa:

- mesmo `group_id` deve resolver o mesmo `schema_name`;
- schema existente é aceito somente se estiver registrado para o grupo;
- versão e checksums já aplicados são conferidos;
- divergência estrutural interrompe o processo;
- novo tenant sempre termina na versão tenant mais recente.

## Estratégia de migrations

### Versionamento

- arquivos usam números sequenciais e nomes descritivos;
- migrations publicadas são imutáveis;
- checksum será registrado junto da versão aplicada;
- migrations de `platform` e `tenant` têm sequências independentes;
- cada tenant mantém sua tabela local de versões;
- `groups.current_schema_version` é atualizado somente após sucesso;
- migrations fora de ordem são proibidas.

### Execução

```text
adquirir advisory/session lock global
  -> validar versão do migrador
  -> migrar platform
  -> listar tenants abaixo da versão alvo
  -> marcar lote como migration_pending
  -> migrar tenants com concorrência limitada
  -> validar cada tenant
  -> registrar sucesso ou falha individual
  -> produzir relatório final
  -> liberar lock
```

Concorrência inicial: até quatro tenants, configurável para menos. O limite será revisto por métricas de lock, I/O e duração.

Comandos mínimos:

```text
migrate status
migrate platform up
migrate tenants plan
migrate tenants up --batch-size N
migrate tenant up --group-id UUID
provision-tenant --group-id UUID
verify-schema --group-id UUID
```

`plan` e `status` são somente leitura. Nenhum segredo será aceito como argumento de linha de comando.

### Compatibilidade de deploy

Mudanças seguirão expand/contract:

1. adicionar estrutura compatível e nullable ou com default seguro;
2. publicar código capaz de operar durante a transição;
3. executar backfill em job controlado e retomável;
4. validar dados;
5. ativar nova leitura/escrita;
6. remover estrutura antiga somente em deploy posterior.

Migrations não executarão backfills grandes dentro de uma única transação. Alterações potencialmente bloqueantes exigirão análise de lock e plano de execução. `CREATE INDEX CONCURRENTLY` será migration não transacional isolada.

### Falha e recuperação

- migration transacional falha por inteiro;
- migration não transacional registra passo e procedimento de reparo;
- tenant que falhar recebe `migration_failed` e preserva a última versão válida;
- outros tenants continuam;
- a causa é registrada sem SQL sensível ou dados fiscais;
- reexecução ocorre somente após correção ou confirmação de segurança;
- arquivos aplicados nunca são editados para “consertar” produção; cria-se nova migration corretiva.

Rollback automático destrutivo não será regra. A preferência é roll-forward. Um `down` só será oferecido quando comprovadamente seguro e testado; perda de coluna, tabela, documento, artefato ou histórico exige backup validado e autorização explícita.

### Verificação de drift

O comando `verify-schema` comparará:

- versões e checksums;
- tabelas e colunas esperadas;
- tipos e nullability;
- PKs, FKs, checks e unicidades;
- índices essenciais;
- privilégios e owner;
- ausência de objetos inesperados em schemas críticos.

Drift não será corrigido silenciosamente. Ele bloqueia a migration daquele tenant e gera evidência para análise.

## Retenção, descarte e imutabilidade

O modelo físico suportará as políticas da Etapa 8:

- XML autorizado, protocolo, snapshot fiscal e respostas relevantes: seis anos ou prazo superior aplicável;
- payload original: 180 dias após estado final, salvo `legal_hold`;
- auditoria de segurança: cinco anos;
- metadados de certificado: cinco anos ou prazo fiscal superior;
- PFX substituído: eliminação criptográfica após a janela definida;
- exclusão lógica para grupos, empresas, integrações, chaves e configurações usadas;
- `legal_hold` impede expiração ou destruição normal.

Jobs de retenção trabalharão em lotes, registrarão contagens e não apagarão registros com dependências fiscais. A operação completa será detalhada na Etapa 10.

## Backup e restauração

Requisitos físicos:

- PITR do PostgreSQL com RPO de até 15 minutos;
- backups criptografados e separados das credenciais de runtime;
- vínculo preservado entre banco, referência do objeto, XML e protocolo;
- restauração testada antes da primeira produção;
- restauração nunca sobrescreve produção durante teste;
- schema isolado poderá ser restaurado por procedimento controlado, mas o backup principal continua sendo do banco;
- restauração precisa revalidar versões, hashes dos artefatos e integridade referencial.

RTO geral inicial: até quatro horas. O XML confirmado precisa de RPO próximo de zero no armazenamento de objetos; autorização não é concluída para o cliente antes da persistência segura do XML e protocolo.

## Estratégia de testes

### Estrutura e migrations

- criar banco vazio e migrar até a última versão;
- executar novamente e confirmar ausência de mudanças;
- validar checksums e ordem;
- testar cada migration suportada a partir da versão anterior;
- provisionar tenant diretamente na versão atual;
- verificar drift provocado deliberadamente;
- simular falha transacional e não transacional;
- retomar tenant com falha sem afetar outro;
- validar permissões depois de criar novos objetos.

### Multi-tenancy

- provisionar ao menos dois tenants;
- repetir IDs de negócio em schemas diferentes sem colisão;
- impedir acesso com `company_id`, `integration_id`, documento e certificado de outro tenant;
- rejeitar `schema_name` em payload, header, fila ou parâmetro;
- reutilizar conexões do pool e confirmar ausência de vazamento de `search_path`;
- confirmar que papéis de runtime não criam ou alteram schemas;
- suspender grupo e confirmar bloqueio imediato.

### Concorrência e integridade

- reservar números simultaneamente sem duplicidade;
- repetir idempotency key com mesma fingerprint;
- repetir chave com fingerprint diferente e obter conflito;
- disputar processamento do mesmo documento com vários workers;
- recuperar lease expirada sem retransmissão cega;
- atualizar estado e histórico atomicamente;
- rotacionar e resetar API keys sob concorrência;
- publicar outbox sem perder ou duplicar efeito de negócio.

### Segurança e dados

- confirmar ausência de PFX, senha e API key em texto puro;
- detectar alteração em ciphertext e artefato pelo hash;
- negar `UPDATE` e `DELETE` em históricos imutáveis;
- executar expiração com e sem `legal_hold`;
- restaurar backup em ambiente isolado;
- verificar que erros de migration não revelam schema ou dados ao cliente.

### Desempenho inicial

- medir índices das consultas principais com `EXPLAIN (ANALYZE, BUFFERS)` usando dados sintéticos;
- medir fila e leases concorrentes;
- medir migration em lote de schemas sintéticos;
- validar limites de payload e crescimento de JSONB;
- registrar baseline, sem antecipar particionamento.

## Critérios para concluir a implementação da Etapa 9

O planejamento fica concluído com este documento. A implementação somente poderá mover a etapa para `docs/implementado` quando houver:

1. migrations `platform` e `tenant` versionadas;
2. comando de migration com lock, status, plano, lote e execução individual;
3. provisionamento idempotente de tenant;
4. papéis e privilégios aplicados;
5. todos os constraints e índices essenciais;
6. testes automatizados de isolamento, idempotência, numeração e leases;
7. teste de falha e retomada de migration;
8. teste de novo tenant na versão atual;
9. verificação de drift;
10. prova em PostgreSQL descartável suportado;
11. restauração documentada e testada antes da produção;
12. revisão de segurança para permissões, schema selection e dados cifrados.

## Decisões adiadas conscientemente

- particionamento de tabelas por data;
- sharding ou múltiplos bancos;
- RLS dentro de cada schema de tenant;
- replicação analítica;
- data warehouse;
- escolha definitiva de broker externo;
- arquivamento em camada fria;
- restauração self-service por tenant;
- migrations online automatizadas para qualquer tipo de DDL;
- banco dedicado por cliente.

Essas capacidades serão avaliadas por volume, risco, contrato e operação real. O modelo não depende delas para o MVP.

## Decisões consolidadas da Etapa 9

1. O banco será PostgreSQL 18, na versão minor mais recente, em todos os ambientes.
2. O acesso Go usará `pgx/v5`; migrations serão SQL-first com Goose e comando próprio.
3. `platform` e `tenant` terão trilhas independentes de migrations.
4. UUID v7 será gerado pela aplicação e armazenado como `uuid`.
5. Valores fiscais usarão `numeric`; ponto flutuante é proibido.
6. Status usarão `text` com checks, não enums físicos.
7. JSONB será limitado a estruturas versionadas e não substituirá invariantes relacionais.
8. XML ficará em armazenamento privado; o banco guardará referência, hash e retenção.
9. Certificado ficará cifrado no schema do tenant por criptografia envelope.
10. Numeração será reservada por atualização atômica no banco.
11. Idempotência terá índice único no escopo do tenant e fingerprint canônica.
12. Estado atual e histórico serão gravados na mesma transação.
13. Processamento assíncrono usará outbox transacional e leases.
14. O contexto do tenant existirá somente em transação com `SET LOCAL search_path`.
15. Runtime não possuirá permissão de DDL ou criação de schemas.
16. Migrations usarão lock global, lotes limitados, histórico por tenant e recuperação individual.
17. Deploys de banco seguirão expand/contract e preferência por roll-forward.
18. Drift será detectado e bloqueará correção automática.
19. Novos tenants nascerão na versão mais recente.
20. Um tenant parcialmente provisionado ou incompatível nunca ficará ativo.

## Resultado esperado

A implementação terá um modelo físico suficientemente definido para iniciar o MVP sem decisões estruturais pendentes: isolamento por schema, autenticação central, dados fiscais versionados, certificados protegidos, idempotência e numeração garantidas pelo banco, processamento rastreável e evolução segura de todos os tenants.

O próximo planejamento poderá tratar operação e suporte sabendo quais estados, filas, históricos, auditorias e sinais físicos estarão disponíveis.
