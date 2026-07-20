# Multi-tenancy: grupos, schemas e empresas

## Objetivo

Organizar os clientes da plataforma com isolamento de dados por schema no Postgres.

Cada cliente da API será representado por um **grupo**, também chamado de tenant. Cada grupo terá exatamente um schema exclusivo, e cada schema pertencerá a somente um grupo.

Dentro do schema do grupo ficarão todas as empresas emitentes e todos os dados relacionados a elas.

```text
Grupo A -> schema exclusivo do Grupo A
Grupo B -> schema exclusivo do Grupo B
Grupo C -> schema exclusivo do Grupo C
```

Um grupo poderá possuir uma ou várias empresas.

## Regra principal

```text
Um grupo possui exatamente um schema.
Um schema pertence a exatamente um grupo.
Um grupo pode possuir várias empresas.
Uma empresa pertence a somente um grupo.
```

Exemplo:

```text
Grupo Jorge
├── Loja Jorge Matriz
├── Loja Jorge Filial
└── Distribuidora Jorge
```

As três empresas ficam dentro do mesmo schema do Grupo Jorge. Empresas de outros grupos ficam em outros schemas.

## Identificação do grupo e do schema

O nome comercial do grupo não será usado para formar o nome do schema, pois nomes podem mudar ou se repetir.

Na criação do grupo, o sistema gerará um UUID exclusivo. O nome físico do schema será derivado desse identificador, removendo caracteres que exigiriam tratamento especial no Postgres.

Exemplo:

```text
group_id:    019f7a12-8c40-7b21-9d32-4a915f21a022
schema_name: tenant_019f7a128c407b219d324a915f21a022
```

O banco deverá garantir a unicidade do `group_id` e do `schema_name`:

```text
groups.id          -> PRIMARY KEY
groups.schema_name -> UNIQUE
```

O UUID torna uma colisão impraticável, e as restrições do banco impedem que dois grupos sejam persistidos com o mesmo identificador ou schema.

O nome do schema sempre será gerado pela aplicação. Ele nunca será recebido do cliente por payload, header, parâmetro ou query string.

## Organização do banco de dados

O banco será dividido entre um schema central da plataforma e os schemas exclusivos dos grupos.

```text
Postgres
│
├── platform
│   ├── groups
│   └── api_keys
│
├── tenant_<uuid_grupo_a>
│   ├── companies
│   ├── integrations
│   ├── fiscal_document_requests
│   ├── fiscal_documents
│   ├── fiscal_document_items
│   ├── products
│   ├── recipients
│   └── tax_rules
│
└── tenant_<uuid_grupo_b>
    └── mesmas tabelas, com dados separados
```

## Schema central da plataforma

O schema `platform` guardará somente as informações necessárias para autenticar, localizar e administrar os grupos.

### groups

```text
id
name
schema_name
status
created_at
updated_at
```

Status iniciais sugeridos:

```text
provisioning
active
suspended
cancelled
error
```

Um grupo que não esteja `active` não poderá consumir a API, mesmo que possua uma API Key ativa.

### api_keys

```text
id
group_id
prefix
secret_hash
status
last_used_at
created_at
revoked_at
```

A API Key permanecerá no schema central porque precisa ser validada antes de a aplicação saber qual schema acessar.

No MVP, cada grupo terá somente uma API Key ativa, conforme definido no planejamento de autenticação.

## Schema exclusivo do grupo

O schema do grupo concentrará os dados do negócio pertencentes àquele tenant.

### Empresas

A tabela `companies` armazenará as empresas emitentes do grupo.

```text
companies
- id
- cnpj
- razao_social
- nome_fantasia
- inscricao_estadual
- uf
- regime_tributario
- status
- created_at
- updated_at
```

Cada empresa terá seu próprio `company_id`. Todas as notas e configurações fiscais deverão manter o vínculo com a empresa emitente correspondente.

Embora o schema já identifique o grupo, o `company_id` continua obrigatório para diferenciar as empresas que existem dentro daquele grupo.

### Integrações

A tabela `integrations` armazenará as formas pelas quais os sistemas do grupo consomem a API.

```text
integrations
- id
- name
- input_type
- mapping_config
- environment
- status
- created_at
- updated_at
```

O `input_type` poderá ser `standard` ou `custom`. Uma integração padrão seguirá o contrato oficial da API. Uma integração customizada terá um mapeamento próprio para transformar o payload recebido no modelo interno.

Como as integrações ficam dentro do schema do grupo, elas não poderão ser utilizadas por outro tenant.

## Criação de um grupo

A criação do grupo deverá ocorrer nesta ordem:

1. Gerar o UUID do grupo.
2. Derivar o nome do schema usando o UUID.
3. Registrar o grupo em `platform.groups` com status `provisioning`.
4. Criar o schema exclusivo do grupo.
5. Aplicar a estrutura inicial de tabelas e índices nesse schema.
6. Alterar o status do grupo para `active` quando a criação for concluída.

A operação deverá ser transacional. Se o schema ou alguma tabela não puder ser criada, todo o processo será desfeito. O sistema não deverá manter um grupo criado parcialmente.

## Cadastro das empresas e integrações

Depois que o grupo estiver ativo, a equipe administrativa poderá cadastrar uma ou várias empresas dentro do schema dele.

Em seguida, serão cadastradas as integrações usadas pelo grupo, definindo se seguem o formato padrão ou um mapeamento customizado.

O onboarding seguirá esta ordem:

```text
criar grupo
-> criar schema
-> cadastrar empresas
-> cadastrar integrações
-> gerar API Key
-> entregar credencial ao responsável técnico
```

A API Key somente deverá ser entregue depois que o cadastro mínimo do grupo estiver concluído.

## Processamento de uma requisição

Quando um ERP chamar a API, o fluxo será:

1. Receber a API Key no cabeçalho de autenticação.
2. Localizar e validar a credencial em `platform.api_keys`.
3. Identificar o `group_id` vinculado à chave.
4. Confirmar que a chave e o grupo estão ativos.
5. Obter o `schema_name` registrado em `platform.groups`.
6. Iniciar uma transação usando o schema exclusivo do grupo.
7. Localizar a empresa informada pelo `company_id` dentro desse schema.
8. Confirmar que a empresa e a integração estão ativas.
9. Executar a operação somente dentro do schema selecionado.

```text
API Key
-> grupo
-> schema exclusivo
-> integração
-> empresa emitente
-> operação
```

O cliente poderá informar o `company_id` e, quando necessário, o `integration_id`. O `group_id` e o `schema_name` nunca serão aceitos como fonte de confiança na requisição; eles sempre serão obtidos internamente a partir da API Key.

## Isolamento entre grupos

Se uma chave pertence ao Grupo A, a aplicação acessará apenas o schema do Grupo A.

Caso o cliente envie um `company_id` de outro grupo, essa empresa não será encontrada no schema selecionado e a operação será recusada. A aplicação não deverá procurar o identificador em outros schemas.

Ao usar pool de conexões, a seleção do schema deverá ocorrer no escopo da transação, usando uma configuração local como `SET LOCAL search_path`. Isso evita que uma conexão reutilizada mantenha o schema de uma requisição anterior.

O identificador utilizado na seleção do schema deve vir apenas de `platform.groups`, com formato controlado pela aplicação. Valores informados pelo cliente nunca poderão ser interpolados em comandos SQL.

## API Keys e reset

A API Key pertence ao grupo, não a uma empresa específica. No MVP, ela permitirá acesso às empresas ativas existentes dentro daquele grupo.

Quando uma chave for resetada:

1. Todas as chaves ativas do grupo serão revogadas.
2. Uma nova chave será criada e vinculada ao mesmo `group_id`.
3. O schema e todos os dados do grupo permanecerão inalterados.
4. Somente a nova chave poderá acessar a API.

O gerenciamento completo das chaves está documentado em `AUTENTICACAO_E_GERENCIAMENTO_DE_API_KEYS.md`.

## Atualização da estrutura dos schemas

Todos os schemas de grupos deverão possuir a mesma versão estrutural.

Quando uma nova tabela, coluna ou índice for necessário, a migration deverá ser aplicada em todos os schemas existentes. A versão de cada schema deverá ser registrada para permitir acompanhamento, repetição segura e recuperação em caso de falha.

Novos grupos deverão ser criados diretamente com a versão estrutural mais recente.

## Exclusão e desativação

Grupos, empresas e integrações que já possuam movimentação fiscal não deverão ser excluídos fisicamente.

Um grupo suspenso ou cancelado continuará com seu schema e histórico preservados, mas não poderá consumir a API. Empresas e integrações desativadas também permanecerão registradas para auditoria e rastreabilidade.

## Decisão arquitetural

A arquitetura multi-tenant adotará isolamento por schema no Postgres:

```text
um grupo por schema
um schema por grupo
várias empresas por grupo
todos os dados fiscais e operacionais dentro do schema do grupo
autenticação e localização dos grupos no schema central platform
```

A API Key identifica o grupo, o grupo determina o schema e o `company_id` identifica a empresa emitente dentro daquele tenant.
