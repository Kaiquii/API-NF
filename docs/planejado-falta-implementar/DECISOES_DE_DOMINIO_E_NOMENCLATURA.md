# Decisoes de dominio e nomenclatura

Este documento e a referencia para nomes tecnicos e conceitos de dominio usados pela plataforma. Todos os documentos, contratos de API, tabelas e modulos futuros devem seguir estas definicoes.

## Grupo

`group` e a organizacao cliente da plataforma, tambem chamada de tenant. Ele e a fronteira de isolamento de dados e de acesso.

Um grupo possui exatamente um schema exclusivo no Postgres e pode possuir varias empresas emitentes.

Nomes oficiais:

```text
group
group_id
platform.groups
platform.api_keys.group_id
```

O termo `client` pode ser usado apenas em comunicacao comercial ou para se referir a quem consome a API. Ele nao deve ser usado como entidade, tabela, rota administrativa ou campo tecnico.

## Empresa

`company` e uma empresa emitente pertencente a um grupo. Ela representa o CNPJ, inscricoes, regime tributario, configuracoes fiscais, certificados e numeracao usados para emitir documentos.

Nomes oficiais:

```text
company
company_id
companies
```

Uma API Key autoriza o grupo. A escolha da empresa emissora ocorre por `company_id`, sempre dentro do schema do grupo autenticado.

## Integracao

`integration` representa uma forma configurada de entrada de dados de um grupo. Ela pode seguir o contrato padrao da plataforma ou possuir mapeamento customizado.

Nomes oficiais:

```text
integration
integration_id
integrations
```

Uma integracao pertence ao schema do grupo e nunca pode ser reutilizada por outro grupo.

## Documento fiscal

`fiscal document` e o nome generico do recurso emitido pela plataforma. O termo `invoice` nao deve ser usado como nome generico, pois a plataforma pode emitir documentos que nao sao NF-e.

Nomes oficiais:

```text
fiscal_document
fiscal_document_id
fiscal_document_request
fiscal_document_requests
fiscal_documents
fiscal_document_items
```

Cada documento possui um `document_type`, como `nfe`, `nfce`, `nfse`, `cte` ou `mdfe`. Cada tipo possui contrato interno, regras, XML e conector proprios.

## Autoridade fiscal

`fiscal authority` e o termo generico para o destino de autorizacao fiscal. O nome especifico, como SEFAZ ou outro orgao responsavel, deve ser usado apenas dentro do modulo do documento quando aplicavel.

Isso evita que a plataforma seja planejada como se todos os documentos fossem transmitidos ao mesmo servico.

## Convencoes de API

Rotas externas devem usar `fiscal-documents` como recurso generico. O grupo e identificado exclusivamente pela API Key e nao aparece como parametro de rota ou corpo da requisicao.

Exemplos de nomenclatura:

```http
POST /v1/fiscal-documents
POST /v1/integrations/{integration_id}/fiscal-documents
GET /v1/fiscal-documents/{fiscal_document_id}
GET /v1/fiscal-documents/{fiscal_document_id}/xml
```

Rotas administrativas devem usar `groups`:

```http
POST /admin/v1/groups/{group_id}/api-keys
GET /admin/v1/groups/{group_id}/api-keys
POST /admin/v1/groups/{group_id}/api-keys/reset
```

## Regra de migracao da documentacao

Documentos anteriores que usem `client`, `client_id`, `clients`, `invoice` ou `invoices` como conceitos genericos devem ser lidos conforme esta referencia. Eles devem ser atualizados antes da implementacao do modulo, tabela ou endpoint correspondente.
