# Planejamento inicial da API de Nota Fiscal

## Objetivo

Criar uma API em Go, usando Postgres como banco de dados, para receber dados de documentos fiscais enviados pelos grupos, transformar esses dados em um modelo interno padronizado, gerar o documento fiscal e transmitir para a autoridade fiscal responsavel.

A ideia principal e que os clientes possam consumir nossa API mesmo que seus sistemas enviem dados em formatos diferentes. Por isso, a API precisa ter uma camada de entrada flexivel, mas manter um modelo interno rigido e bem validado antes de gerar qualquer XML fiscal.

## Visao geral da arquitetura

```text
Sistema do cliente
        |
        v
API de entrada / Ingestao
        |
        v
Mapeamento e normalizacao
        |
        v
Modelo interno padrao da nota
        |
        v
Enriquecimento de dados fiscais
        |
        v
Validacao comercial e fiscal
        |
        v
Geracao de XML
        |
        v
Assinatura digital
        |
        v
Envio para autoridade fiscal
        |
        v
Consulta de processamento
        |
        v
Retorno para o cliente
```

## Ideia principal

A API nao deve obrigar todos os clientes a enviarem os dados exatamente no mesmo formato logo no primeiro momento. Muitos clientes terao sistemas legados ou ERPs com estruturas proprias.

Por isso, vamos trabalhar com duas camadas:

1. Entrada flexivel, onde cada cliente pode enviar os dados no formato dele.
2. Modelo interno padrao, usado pela nossa aplicacao para validar, gerar XML, assinar e enviar para a autoridade fiscal responsavel.

Exemplo:

```json
{
  "produto": "Mouse",
  "preco": "50,00"
}
```

ou:

```json
{
  "items": [
    {
      "description": "Mouse",
      "unit_price": 50
    }
  ]
}
```

Os dois formatos devem ser transformados internamente para algo como:

```json
{
  "items": [
    {
      "description": "Mouse",
      "quantity": 1,
      "unit_amount": 50.00,
      "ncm": null,
      "cfop": null
    }
  ]
}
```

## Camadas sugeridas no projeto Go

```text
/cmd/api
  Servidor HTTP principal.

/cmd/worker
  Processamento em background das notas.

/internal/ingestion
  Recebe payloads em formatos diferentes.

/internal/mapping
  Converte o payload externo do cliente para o modelo interno.

/internal/normalization
  Normaliza documentos, valores, datas, unidades e campos textuais.

/internal/enrichment
  Completa dados faltantes usando cadastros internos.

/internal/validation
  Valida dados comerciais e fiscais antes da emissao.

/internal/fiscaldocument
  Modelo interno e regras de negocio compartilhadas dos documentos fiscais.

/internal/fiscalauthority
  Comunicacao com autoridades fiscais, assinatura, envio e consulta.

/internal/certificate
  Leitura e uso de certificado digital A1.

/internal/database
  Acesso ao Postgres.

/internal/danfe
  Geracao futura do DANFE em PDF.
```

## Fluxo de emissao

1. Cliente envia um payload para nossa API.
2. API identifica o cliente pela chave de API ou credencial.
3. Payload original e salvo no Postgres.
4. Sistema aplica o mapeamento configurado para aquele cliente.
5. Payload e convertido para o modelo interno da nota.
6. Sistema normaliza dados como CPF, CNPJ, valores e datas.
7. Sistema enriquece a nota com dados cadastrados, como NCM, CFOP, CST, CSOSN, unidade e regras fiscais.
8. Sistema valida se a nota esta pronta para emissao.
9. Se faltar informacao, a nota fica com status de erro ou pendencia.
10. Se estiver valida, a nota entra na fila de emissao.
11. Worker gera o XML da NF-e ou NFC-e.
12. Worker assina o XML com certificado digital A1.
13. Worker envia o lote para a autoridade fiscal responsavel.
14. Worker consulta o processamento.
15. Sistema salva protocolo, chave de acesso, XML autorizado e status final.
16. Cliente consulta a nota ou recebe retorno/webhook.

## Endpoints iniciais

### Entrada no formato padrao da nossa API

```http
POST /v1/fiscal-documents
```

Usado por clientes que conseguem se adequar ao contrato recomendado da nossa API.

### Entrada por integracao configurada

```http
POST /v1/integrations/{integration_id}/fiscal-documents
```

Usado por clientes que enviam dados em um formato proprio. Nesse caso, a API usa a configuracao de mapeamento da integracao.

### Consulta de nota

```http
GET /v1/fiscal-documents/{id}
```

Retorna status, chave de acesso, protocolo, erros e dados principais da nota.

### Download do XML

```http
GET /v1/fiscal-documents/{id}/xml
```

Retorna o XML assinado/autorizado.

### Cancelamento

```http
POST /v1/fiscal-documents/{id}/cancel
```

Cancela uma nota ja autorizada, quando permitido pela regra fiscal.

### Reprocessamento

```http
POST /v1/fiscal-documents/{id}/retry
```

Tenta reprocessar uma nota que falhou por erro temporario.

## Tabelas iniciais no Postgres

### platform.groups e platform.api_keys

O schema central da plataforma guarda os grupos que consomem a API e suas credenciais. Os dados fiscais e operacionais ficam apenas no schema exclusivo de cada grupo.

```text
groups: id, name, schema_name, status, created_at, updated_at
api_keys: id, group_id, prefix, secret_hash, status, created_at, updated_at
```

### companies

Empresas emitentes das notas.

```text
id
cnpj
razao_social
nome_fantasia
inscricao_estadual
uf
ambiente
regime_tributario
certificado_encrypted
certificado_password_encrypted
created_at
updated_at
```

### integrations

Configuracoes de entrada e mapeamento por cliente.

```text
id
name
input_type
mapping_config jsonb
active
created_at
updated_at
```

Exemplo de `mapping_config`:

```json
{
  "customer.name": "destinatario.nome",
  "customer.document": "destinatario.cpf_cnpj",
  "products[].title": "itens[].descricao",
  "products[].price": "itens[].valor_unitario",
  "products[].qty": "itens[].quantidade"
}
```

### fiscal_document_requests

Registra tudo que chegou na API antes e depois da normalizacao.

```text
id
integration_id
raw_payload jsonb
normalized_payload jsonb
status
validation_errors jsonb
created_at
updated_at
```

### fiscal_documents

Representa a nota fiscal no nosso sistema.

```text
id
company_id
fiscal_document_request_id
numero
serie
modelo
status
chave_acesso
protocolo
recibo
motivo_rejeicao
total
xml_assinado
xml_autorizado
created_at
updated_at
```

### fiscal_document_items

Itens da nota fiscal.

```text
id
fiscal_document_id
sku
descricao
ncm
cfop
cst
csosn
unidade
quantidade
valor_unitario
valor_total
created_at
updated_at
```

### products

Cadastro auxiliar para completar informacoes fiscais quando o cliente envia apenas SKU ou dados basicos.

```text
id
company_id
sku
descricao
ncm
cfop_padrao
unidade
origem
cst
csosn
active
created_at
updated_at
```

### customers

Cadastro de destinatarios.

```text
id
cpf_cnpj
nome
inscricao_estadual
indicador_ie
email
telefone
endereco_json
created_at
updated_at
```

### tax_rules

Regras fiscais para completar dados da emissao.

```text
id
company_id
uf_origem
uf_destino
regime_tributario
tipo_operacao
cfop
cst
csosn
aliquotas_json
active
created_at
updated_at
```

## Status sugeridos

```text
received
mapped
pending_data
validating
ready_to_issue
queued
processing
authorized
rejected
cancelled
error
```

## Informacoes que podem faltar no payload do cliente

O cliente pode enviar poucos dados, mas a NF-e precisa de informacoes fiscais obrigatorias. Por isso, nossa API deve conseguir completar dados usando cadastros internos.

Campos que provavelmente precisaremos cadastrar ou inferir:

- NCM
- CFOP
- CST ou CSOSN
- unidade comercial
- origem da mercadoria
- regime tributario do emitente
- inscricao estadual
- endereco do destinatario
- regras de ICMS, PIS, COFINS e IPI
- serie e numeracao da nota
- certificado digital

## Estrategia de produto

Vamos oferecer um contrato oficial recomendado, mas tambem permitir integracoes customizadas.

Formato recomendado:

```http
POST /v1/fiscal-documents
```

Formato flexivel:

```http
POST /v1/integrations/{integration_id}/fiscal-documents
```

Assim, clientes mais organizados podem consumir diretamente o contrato padrao, enquanto clientes com sistemas legados podem usar uma integracao configurada.

## MVP recomendado

Para comecar com seguranca:

1. Emitir NF-e modelo 55.
2. Usar ambiente de homologacao.
3. Comecar com um unico emitente.
4. Comecar com um unico estado.
5. Usar certificado digital A1.
6. Receber payload JSON.
7. Salvar payload original e payload normalizado.
8. Criar mapeamento configuravel por cliente.
9. Validar dados obrigatorios.
10. Gerar XML.
11. Assinar XML.
12. Enviar para a autoridade fiscal responsavel.
13. Consultar autorizacao.
14. Salvar XML autorizado.

Depois do MVP, podemos evoluir para:

- multiplos emitentes;
- NFC-e modelo 65;
- cancelamento;
- inutilizacao de numeracao;
- carta de correcao;
- contingencia;
- DANFE em PDF;
- webhooks para avisar status ao cliente;
- painel administrativo para configurar mapeamentos.

## Decisao arquitetural importante

A camada da autoridade fiscal nunca deve conhecer o formato original recebido do grupo.

O cliente pode mandar dados em varios formatos, mas antes de chegar na emissao fiscal tudo precisa estar convertido para um modelo interno unico, rigido, rastreavel e validado.

Essa separacao evita que o sistema fiscal vire um conjunto de excecoes por cliente e facilita manutencao, suporte e evolucao do produto.
