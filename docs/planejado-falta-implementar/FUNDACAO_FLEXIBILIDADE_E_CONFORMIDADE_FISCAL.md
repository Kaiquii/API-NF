# Fundacao: flexibilidade e conformidade fiscal

## Decisao central

A plataforma deve ser flexivel para receber, organizar e apresentar os dados ao usuario, mas rigida para validar e emitir documentos fiscais.

A configuracao de cada grupo ou empresa pode adaptar a experiencia de preenchimento e o formato de entrada. Nenhuma configuracao pode alterar exigencias legais, regras fiscais, calculos, XML, assinatura ou integracao com a autoridade fiscal.

```text
Entrada configuravel
  -> mapeamento e normalizacao
  -> modelo fiscal interno oficial
  -> validacao legal e fiscal
  -> geracao e assinatura do XML
  -> transmissao e retorno da autoridade fiscal
```

## Limites da flexibilidade

### O que pode ser configurado

- formularios e ordem de apresentacao dos campos;
- rotulos e nomes comerciais dos campos;
- campos extras nao fiscais;
- obrigatoriedade comercial de campos extras;
- valores padrao;
- mapeamento de payloads externos para campos internos;
- visibilidade condicional de campos na interface.

Exemplo: um cliente pode chamar o codigo do produto de `sku`, `codigoProduto` ou `produto.codigo`. Todos devem ser mapeados para o mesmo campo interno, `item.product_code`.

### O que nao pode ser configurado livremente

- definicao e tipo de dado dos campos fiscais internos;
- campos obrigatorios por exigencia legal;
- calculos de impostos e totais;
- regras tributarias e validacoes fiscais;
- schema e estrutura do XML fiscal;
- assinatura digital;
- regras de comunicacao, autorizacao e eventos da autoridade fiscal.

O cliente pode informar um dado fiscal, como CFOP, quando a integracao permitir. A plataforma sempre deve validar se ele e compativel com a empresa, operacao e regras fiscais vigentes.

## Nucleo fiscal interno

Todos os dados recebidos devem ser convertidos para um documento fiscal interno antes de qualquer tentativa de emissao.

O fluxo compartilhado por todos os documentos deve usar um envelope comum:

```json
{
  "id": "uuid",
  "document_type": "nfe",
  "document_version": "1",
  "company_id": "uuid",
  "operation": "sale",
  "data": {}
}
```

O campo `data` deve seguir um contrato interno proprio, tipado e versionado para cada documento. A plataforma nao deve criar um unico modelo universal que tente representar todos os documentos fiscais.

Exemplos de contratos internos:

```text
nfe/v1
nfce/v1
nfse/v1
cte/v1
mdfe/v1
```

Assim, autenticacao, processamento, auditoria e notificacoes permanecem compartilhados, enquanto regras, XML e validacoes especificas permanecem isolados em cada modulo fiscal.

## Politica de validacao

A emissao deve passar por tres niveis de validacao:

1. Entrada: formato, tipos de dados, campos minimos e resultado do mapeamento.
2. Fiscal: cadastros, regras tributarias, calculos, obrigatoriedades e consistencia da operacao.
3. Emissao: schema XML, assinatura e resposta da autoridade fiscal.

Erros fiscais ou legais bloqueiam a emissao. Avisos comerciais podem existir, desde que sejam explicitos e nao escondam uma pendencia fiscal.

A plataforma nao deve transmitir um documento apenas para verificar se sera aceito. A validacao interna deve impedir tentativas sabidamente invalidas.

## Responsabilidade e origem dos dados

Cada campo relevante deve ter origem e precedencia definidas, registradas junto da nota para auditoria.

| Origem | Responsabilidade |
| --- | --- |
| Usuario ou ERP | Dados da venda, itens, destinatario e intencao da operacao. |
| Cadastros | Produto, NCM, unidade, emitente e destinatario recorrente. |
| Regras fiscais | CFOP permitido, tributacao, CST/CSOSN e aliquotas. |
| Motor fiscal | Bases de calculo, impostos, totais, chave e XML. |

Um dado enviado pelo cliente somente pode sobrescrever um cadastro quando a politica daquele campo permitir. Nenhum complemento que afete tributacao ou valores pode ocorrer sem registrar sua origem e regra aplicada.

## Versionamento e atualizacao de regras

Regras fiscais devem ser organizadas em pacotes versionados, imutaveis e com periodo de vigencia.

Cada documento emitido deve guardar, no minimo:

- versao do contrato interno utilizado;
- versao do modulo fiscal;
- versao do pacote de regras fiscais;
- versao do layout fiscal aplicado;
- data e hora da decisao fiscal.

Atualizacoes devem passar por homologacao e testes antes de serem ativadas em producao. Uma nova regra nao deve modificar retrospectivamente o calculo de notas ja emitidas.

## Modulos de documentos fiscais

Cada tipo de documento deve existir como um modulo independente, implementando um contrato tecnico comum:

```text
validar
calcular
gerar XML
assinar
transmitir
consultar status
tratar eventos fiscais
```

Um modulo so pode ser habilitado para uma empresa quando estiver implementado, testado e homologado para aquele ambiente e cenario operacional.

A arquitetura nasce preparada para varios tipos de documento, mas a disponibilidade em producao e controlada por modulo, empresa, ambiente e operacao.

## Consequencia para a experiencia do usuario

A interface pode ser configuravel e amigavel, mas deve sempre orientar o usuario para dados validos. Campos exigidos por uma operacao fiscal devem ser solicitados, regras invalidas devem ser bloqueadas e dados calculados pelo motor fiscal devem ser apresentados de forma rastreavel.

Flexibilidade na entrada nao significa liberdade para emitir fora das regras. O nucleo fiscal e a ultima autoridade antes da transmissao.
