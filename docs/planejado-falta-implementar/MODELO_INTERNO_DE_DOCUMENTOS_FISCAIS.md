# Modelo interno de documentos fiscais

## Objetivo

Definir o contrato interno que representa qualquer documento fiscal depois que os dados recebidos por formularios ou integracoes forem mapeados e normalizados.

O modelo interno e o idioma comum da plataforma. A entrada pode variar por grupo e integracao, mas nenhum payload externo segue diretamente para validacao fiscal, geracao de XML ou transmissao.

```text
Payload externo
  -> mapeamento configurado
  -> normalizacao
  -> modelo interno do documento fiscal
  -> validacao fiscal
  -> modulo fiscal especifico
  -> autoridade fiscal responsavel
```

## Principios

- A plataforma possui um envelope comum para todos os documentos fiscais.
- Cada tipo de documento possui contrato proprio, tipado e versionado.
- O modelo interno nao e um JSON universal que tenta acomodar todos os documentos.
- O payload original e preservado para rastreabilidade, mas o nucleo fiscal opera somente sobre dados normalizados.
- O `group_id` nao faz parte do payload ou contrato externo: ele e obtido internamente pela API Key e pelo schema do grupo.
- A NF-e sera usada como primeira referencia completa de contrato, sem limitar a plataforma a esse tipo de documento.

## Envelope comum

Todos os documentos fiscais devem possuir os seguintes dados de contexto:

```json
{
  "id": "uuid",
  "document_type": "nfe",
  "document_version": "1",
  "company_id": "uuid",
  "integration_id": "uuid",
  "environment": "homologation",
  "operation": "sale",
  "external_reference": "PEDIDO-123",
  "idempotency_key": "chave-enviada-pelo-erp",
  "data": {}
}
```

### Responsabilidade dos campos do envelope

| Campo | Finalidade |
| --- | --- |
| `id` | Identificador interno imutavel do documento fiscal. |
| `document_type` | Tipo do documento e modulo fiscal que deve processa-lo. |
| `document_version` | Versao do contrato interno usada no processamento. |
| `company_id` | Empresa emitente, localizada dentro do schema do grupo autenticado. |
| `integration_id` | Integracao que recebeu ou mapeou a solicitacao; pode ser ausente em fluxos internos. |
| `environment` | Ambiente fiscal aplicavel ao processamento, como homologacao ou producao. |
| `operation` | Intencao comercial da operacao, como venda, devolucao ou transferencia. |
| `external_reference` | Referencia do sistema de origem, como numero do pedido ou venda. |
| `idempotency_key` | Chave usada para impedir emissao duplicada da mesma solicitacao. |
| `data` | Dados especificos do contrato do tipo de documento. |

O envelope identifica o documento e seu contexto. Itens, participantes, impostos, transporte e pagamento pertencem ao contrato especifico dentro de `data`.

## Contratos por modulo

Cada tipo de documento possui contrato interno independente. Exemplos:

```text
nfe/v1
nfce/v1
nfse/v1
cte/v1
mdfe/v1
```

Os modulos compartilham o envelope e o fluxo de processamento, mas mantem regras, campos, validacoes, XML e integracoes proprias.

```text
nfe/v1  -> emitente, destinatario, itens, transporte, pagamentos e impostos
nfce/v1 -> consumidor, itens e pagamentos
nfse/v1 -> prestador, tomador, servicos e retencoes
cte/v1  -> remetente, destinatario, carga, veiculo e percurso
mdfe/v1 -> documentos vinculados, veiculos, condutor e percurso
```

## Referencia inicial: nfe/v1

A NF-e sera o primeiro contrato completo usado como referencia de modelagem. Isso nao define a NF-e como unico documento suportado pela plataforma; ela sera apenas o primeiro exemplo com complexidade suficiente para validar os limites do modelo.

O contrato `nfe/v1` deve ser organizado nos seguintes blocos:

```text
identification
issuer
recipient
items
shipping
payments
additional_information
references
```

O detalhamento de campos fiscais, origem de dados e regras tributarias sera realizado nas proximas etapas de planejamento.

## Tipos e normalizacao

O nucleo fiscal deve trabalhar com valores padronizados. A camada de normalizacao converte o formato recebido, mas nao inventa dados, calcula impostos ou toma decisoes tributarias.

Exemplos:

```text
"50,00"                -> 50.00
"12.345.678/0001-90"   -> 12345678000190
"UN", "unid"          -> UN
"2026-07-19"           -> data padronizada
```

Cada campo do contrato especifico deve definir, no minimo:

- tipo de dado;
- obrigatoriedade;
- formato normalizado;
- limites e consistencias basicas;
- possibilidade de ausencia antes do enriquecimento fiscal.

## Rastreabilidade

Para cada documento, a plataforma deve preservar:

- payload original recebido;
- resultado do mapeamento;
- documento normalizado;
- versao do contrato interno;
- versao do modulo fiscal;
- data e hora de cada transformacao relevante.

Esses registros permitem explicar como um dado enviado pelo usuario se tornou o dado usado na emissao.

## Versionamento e compatibilidade

Contratos internos sao imutaveis depois de publicados. Mudancas que alterem estrutura, significado ou validacao de campos devem criar uma nova versao, como `nfe/v2`.

Documentos ja emitidos permanecem vinculados a versao usada no processamento original. Uma versao nova nao pode reinterpretar ou alterar retroativamente documentos existentes.

## Limites desta etapa

Esta etapa define a estrutura interna dos documentos. Ela nao define ainda:

- origem e precedencia de NCM, CFOP, CST, CSOSN ou aliquotas;
- regras tributarias e calculos de impostos;
- regras por UF, regime tributario ou tipo de operacao;
- numeracao, estados, retentativas ou fila;
- XML, assinatura, transmissao ou contingencia.

Esses assuntos serao planejados em etapas proprias.
