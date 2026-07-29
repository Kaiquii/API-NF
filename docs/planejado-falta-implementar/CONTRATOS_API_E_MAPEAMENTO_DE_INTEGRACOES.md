# Contratos da API e mapeamento de integracoes

## Objetivo

Definir como sistemas externos e a interface da plataforma enviam dados, como formatos customizados sao convertidos para o contrato padrao e como a API responde sem expor detalhes internos do nucleo fiscal.

A flexibilidade deve existir na camada de entrada. Depois do mapeamento, todas as solicitacoes seguem o mesmo fluxo de normalizacao, validacao e processamento.

```text
payload padrao ---------------------+
                                    |
payload customizado -> mapeamento --+-> contrato publico padrao
                                         -> normalizacao
                                         -> modelo fiscal interno
                                         -> validacao
                                         -> processamento
```

## Principios

- A API oferece um contrato padrao recomendado.
- Integracoes customizadas podem enviar payloads em formatos proprios.
- Todo payload customizado deve ser convertido para o contrato padrao antes de chegar ao nucleo fiscal.
- O contrato publico nao e a mesma estrutura usada internamente pelo codigo.
- Mapeamentos sao declarativos, versionados e nao executam codigo do cliente.
- Formularios configuraveis alteram a experiencia de preenchimento, nunca as regras fiscais.
- Validacao pode ser executada sem reservar numero ou transmitir documento.
- A emissao e assincrona e retorna um recurso consultavel.
- Versao da API, contrato fiscal e mapeamento sao controles independentes.

## Modos oficiais de entrada

### Contrato padrao

```http
POST /v1/fiscal-documents
```

Esse endpoint recebe o contrato publico oficial da plataforma. Ele deve ser a opcao recomendada para novas integracoes.

Exemplo conceitual:

```json
{
  "document_type": "nfe",
  "document_version": "1",
  "company_id": "uuid",
  "environment": "homologation",
  "operation": "sale",
  "external_reference": "PEDIDO-123",
  "data": {}
}
```

A chave de idempotencia deve ser enviada no cabecalho:

```http
Idempotency-Key: pedido-123-nfe
```

### Contrato customizado

```http
POST /v1/integrations/{integration_id}/fiscal-documents
```

Esse endpoint recebe o formato configurado para a integracao. A plataforma:

1. valida a autenticacao e o pertencimento da integracao;
2. identifica a versao ativa do mapeamento;
3. preserva o payload original;
4. aplica o mapeamento;
5. produz o contrato publico padrao;
6. segue o mesmo fluxo da entrada padrao.

O `integration_id` precisa existir no schema do grupo autenticado e estar ativo. A aplicacao nunca procura uma integracao em outro tenant.

## Separacao entre representacoes

A plataforma deve manter tres representacoes distintas:

```text
payload externo
  -> DTO publico versionado
  -> modelo fiscal interno versionado
```

### Payload externo

Estrutura enviada pelo sistema de origem. Pode seguir o contrato padrao ou um formato customizado.

### DTO publico

Contrato estavel exposto pela API. Define campos aceitos, tipos, erros e compatibilidade da versao publica.

### Modelo fiscal interno

Representacao usada pelo nucleo fiscal. Possui contrato proprio por modulo, como `nfe/v1`, e nao deve ser exposta diretamente como estrutura publica.

Alteracoes no modelo interno nao podem modificar silenciosamente o contrato publico.

## Motor de mapeamento

### Operacoes permitidas

O `mapping_config` pode oferecer operacoes declarativas e controladas:

- copiar ou renomear campos;
- mover valores entre caminhos;
- percorrer objetos e listas;
- definir valores padrao;
- converter datas, documentos, numeros e valores monetarios;
- concatenar ou separar textos;
- converter codigos usando tabelas configuradas;
- aplicar condicoes simples com operadores permitidos;
- ignorar campos externos sem uso fiscal.

Exemplo conceitual:

```json
{
  "target_contract": "nfe/v1",
  "rules": [
    {
      "source": "pedido.cliente.cpf_cnpj",
      "target": "data.recipient.document",
      "transform": "normalize_tax_id"
    },
    {
      "source": "pedido.produtos[]",
      "target": "data.items[]",
      "children": [
        {
          "source": "codigo",
          "target": "product_code"
        },
        {
          "source": "valor",
          "target": "unit_amount",
          "transform": "decimal"
        }
      ]
    }
  ]
}
```

O formato definitivo da DSL sera detalhado antes de sua implementacao. Esse exemplo registra apenas as capacidades aprovadas.

### Operacoes proibidas

O mapeamento nao pode:

- executar JavaScript, Go ou qualquer outro codigo enviado pelo cliente;
- executar SQL ou comandos do sistema;
- realizar chamadas HTTP;
- acessar arquivos, variaveis de ambiente ou segredos;
- alterar validacoes e calculos fiscais;
- consultar dados de outro grupo;
- criar transformacoes sem registro e versionamento.

### Adaptadores especificos

Uma integracao que nao puder ser atendida pelo motor declarativo pode usar um adaptador especifico.

O adaptador deve fazer parte do codigo da plataforma, ser revisado, versionado, testado e registrado na configuracao da integracao. Ele tambem deve produzir o mesmo DTO publico usado pelos demais fluxos.

## Ciclo de vida dos mapeamentos

Cada mapeamento possui versao imutavel e status:

```text
draft -> testing -> active -> retired
```

### Regras

- `draft`: pode ser editado, mas nao processa emissoes reais;
- `testing`: executa casos de teste sem emitir documentos;
- `active`: pode processar solicitacoes de producao;
- `retired`: nao recebe novas solicitacoes, mas permanece disponivel para auditoria.

Uma versao ativa nao pode ser editada. Qualquer alteracao cria uma nova versao em `draft`.

Somente uma versao pode estar ativa por integracao, ambiente e contrato de destino, salvo quando houver uma estrategia de migracao explicitamente planejada.

Cada documento deve registrar a versao do mapeamento que produziu seu DTO publico.

## Teste de mapeamento

Antes da ativacao, um mapeamento precisa possuir casos de teste com:

- payload de entrada;
- resultado esperado;
- campos obrigatorios esperados;
- erros esperados;
- versao do contrato de destino.

O teste executa mapeamento e normalizacao, mas nao reserva numero, cria tentativa de emissao ou transmite documento.

Uma operacao administrativa de teste pode seguir o formato:

```http
POST /admin/v1/groups/{group_id}/integrations/{integration_id}/mapping-versions/{version}/test
```

Ela exige autenticacao administrativa e nunca aceita API Key de cliente.

## Formularios configuraveis

O formulario exibido ao usuario deve ser construido pela composicao de:

```text
contrato do documento
  + cenario da operacao
  + configuracao da empresa
  + configuracao visual do grupo
  = definicao do formulario
```

### Pode ser configurado

- rotulo e texto de ajuda;
- ordem e agrupamento dos campos;
- visibilidade condicional;
- valores padrao permitidos;
- campos comerciais extras;
- apresentacao visual.

### Nao pode ser configurado livremente

- remocao de campo fiscal obrigatorio;
- tipo de dado fiscal;
- regra de calculo;
- validacao legal;
- estrutura do modelo interno;
- permissao para emitir operacao nao habilitada.

O formulario deve utilizar os mesmos identificadores e validacoes do contrato publico para evitar divergencia entre interface e API.

Rascunhos permanecem como recursos da interface e ficam fora da maquina de estados de emissao. O `fiscal_document` e criado somente no envio final.

## Validacao sem emissao

A plataforma deve permitir validacao antes da emissao:

```http
POST /v1/fiscal-documents/validate
POST /v1/integrations/{integration_id}/fiscal-documents/validate
```

Essa operacao pode executar:

- mapeamento;
- normalizacao;
- validacao estrutural;
- validacao dos campos obrigatorios;
- validacoes fiscais disponiveis para o cenario.

Ela nao pode:

- reservar numeracao;
- gerar tentativa de emissao;
- transmitir documento;
- alterar o estado de um documento fiscal existente.

O resultado da validacao deve informar a versao do contrato, a versao do mapeamento e os erros encontrados.

## Emissao assincrona

Uma solicitacao aceita deve retornar `202 Accepted`:

```json
{
  "id": "uuid",
  "status": "pending",
  "external_reference": "PEDIDO-123",
  "links": {
    "self": "/v1/fiscal-documents/uuid"
  }
}
```

O cabecalho `Location` deve apontar para o recurso criado:

```http
Location: /v1/fiscal-documents/uuid
```

A conexao HTTP nao fica aberta aguardando autorizacao fiscal.

Consultas basicas:

```http
GET /v1/fiscal-documents/{fiscal_document_id}
GET /v1/fiscal-documents/{fiscal_document_id}/xml
```

O acesso sempre e limitado ao grupo autenticado e a uma empresa pertencente ao mesmo schema.

## Formato de erros

Erros devem possuir estrutura estavel e utilizavel tanto por pessoas quanto por sistemas:

```json
{
  "code": "mapping.field_not_found",
  "message": "Nao foi possivel localizar o CPF/CNPJ do destinatario.",
  "field": "recipient.document",
  "source_field": "cliente.cpf_cnpj",
  "request_id": "uuid",
  "details": {}
}
```

### Campos

| Campo | Finalidade |
| --- | --- |
| `code` | Codigo estavel para tratamento programatico. |
| `message` | Explicacao compreensivel do problema. |
| `field` | Campo do contrato publico relacionado ao erro. |
| `source_field` | Campo original, quando o erro surgiu no mapeamento. |
| `request_id` | Identificador de correlacao da chamada. |
| `details` | Metadados adicionais que nao exponham dados sensiveis. |

Uma resposta pode conter uma lista de erros quando varios campos puderem ser avaliados na mesma validacao.

### Codigos HTTP

| Status | Uso |
| --- | --- |
| `400` | JSON, cabecalho ou requisicao malformada. |
| `401` | Credencial ausente ou invalida. |
| `403` | Credencial valida sem permissao para a operacao. |
| `404` | Recurso nao encontrado dentro do tenant autenticado. |
| `409` | Conflito de idempotencia, versao ou estado. |
| `422` | Falha de mapeamento, negocio ou validacao fiscal. |
| `429` | Limite de uso excedido. |
| `500` | Erro interno inesperado. |
| `503` | Dependencia temporariamente indisponivel. |

Codigos tecnicos de erro devem ser documentados e permanecer estaveis dentro da versao da API.

## Versionamentos independentes

A plataforma deve controlar separadamente:

```text
/v1                         -> versao da API publica
document_version: nfe/v1    -> versao do contrato fiscal interno
mapping_version: 3          -> versao do mapeamento da integracao
form_version: 2             -> versao da configuracao do formulario
```

Uma nova versao de mapeamento ou formulario nao exige automaticamente uma nova versao da API. Uma alteracao interna tambem nao pode quebrar clientes da mesma versao publica.

Cada solicitacao processada deve registrar todas as versoes que influenciaram seu resultado.

## Rastreabilidade

Para cada entrada, a plataforma deve preservar:

- identificador da requisicao;
- grupo, empresa e integracao;
- payload original;
- versao do mapeamento;
- resultado do mapeamento;
- DTO publico produzido;
- versao do contrato publico;
- modelo interno normalizado;
- erros e avisos;
- chave de idempotencia e impressao digital;
- data e hora das transformacoes.

Dados sensiveis devem seguir as politicas de criptografia, mascaramento e retencao que serao definidas na etapa de seguranca.

## Resultado da etapa

Esta etapa estabelece:

- entrada padrao e entrada customizada;
- separacao entre payload, DTO publico e modelo interno;
- motor de mapeamento declarativo e seguro;
- adaptadores especificos controlados;
- mapeamentos imutaveis e versionados;
- formularios configuraveis sem alteracao das regras fiscais;
- validacao sem emissao;
- processamento assincrono com recurso consultavel;
- respostas e erros estruturados;
- versionamento independente das camadas.
