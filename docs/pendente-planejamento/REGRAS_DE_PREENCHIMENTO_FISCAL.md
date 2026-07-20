# Regras de preenchimento fiscal

## Objetivo

Definir de onde vem cada informação necessária para formar um documento fiscal, quem é responsável pelo dado, quando a plataforma pode completá-lo e quais validações devem ocorrer antes da emissão.

Esta etapa começa pela NF-e do MVP. A matriz completa de campos ainda será detalhada durante o planejamento.

## Decisão central

A plataforma continuará flexível e amigável na entrada dos dados, mas será rígida na validação e na emissão fiscal.

```text
Entrada flexível
  -> mapeamento por integração
  -> normalização
  -> enriquecimento por cadastros e regras
  -> modelo interno nfe/v1
  -> validação fiscal obrigatória
  -> cálculo, XML, assinatura e transmissão
```

O cliente não precisará conhecer ou montar toda a estrutura fiscal da NF-e. Ele informará os fatos comerciais da operação, e a plataforma usará cadastros e regras versionadas para completar e validar o documento.

Flexibilidade não significa aceitar qualquer valor fiscal. Nenhuma configuração do grupo ou da integração poderá modificar exigências legais, regras tributárias, cálculos, leiaute XML, assinatura ou regras de autorização da SEFAZ.

## Regra de responsabilidade

> O ERP informa o que aconteceu comercialmente; a plataforma decide e valida como isso será representado fiscalmente.

As responsabilidades serão divididas em quatro origens.

### ERP ou usuário

O sistema de origem informa os fatos da operação:

- empresa emitente selecionada;
- tipo e finalidade da operação;
- destinatário;
- produtos ou serviços envolvidos;
- quantidades;
- valores unitários;
- descontos e outras informações comerciais;
- frete e pagamento, quando aplicáveis;
- referência externa, como pedido ou venda;
- chave de idempotência.

A plataforma não deve inventar quantidades, valores de venda, destinatários ou a intenção comercial da operação.

### Cadastros da plataforma

Os cadastros fornecem informações estáveis e reutilizáveis:

- dados cadastrais e fiscais da empresa emitente;
- endereço e inscrições da empresa;
- regime tributário;
- configurações fiscais da empresa;
- descrição, NCM, unidade e origem dos produtos;
- dados de destinatários recorrentes;
- série e configurações de numeração.

Um cadastro pode completar o payload quando essa possibilidade estiver prevista na política do campo.

### Regras fiscais

As regras fiscais determinam ou validam o enquadramento tributário conforme o cenário:

- CFOP;
- CST ou CSOSN;
- alíquotas aplicáveis;
- benefícios, reduções e demais enquadramentos;
- compatibilidade entre operação, produto, emitente e destinatário.

Uma sugestão enviada pelo ERP somente poderá ser aceita depois de validada. O ERP não terá autoridade para alterar livremente a decisão fiscal da plataforma.

### Motor fiscal

O motor fiscal será responsável pelos resultados derivados da operação:

- bases de cálculo;
- valores dos tributos;
- totais do documento;
- chave de acesso;
- estrutura fiscal final usada na geração do XML.

Valores calculados não devem ser aceitos cegamente do ERP. Quando forem recebidos para conferência ou compatibilidade, a plataforma deverá validá-los ou recalculá-los conforme a política definida.

## Recorte inicial do MVP

A primeira matriz de preenchimento será construída para um cenário controlado:

```text
Documento: NF-e modelo 55
Operação inicial: venda
Ambiente inicial: homologação
Emitente: uma empresa inicial
UF: UF da empresa emitente inicial
Regime tributário: regime real da empresa emitente inicial
Destino inicial: operação interna
```

O regime tributário e a UF não serão escolhidos apenas para simplificar o desenvolvimento. Eles deverão corresponder à empresa real usada na homologação.

Depois que esse cenário estiver completamente planejado, implementado e validado, a matriz poderá ser ampliada para operações interestaduais, devoluções, transferências, outros regimes e outros documentos fiscais.

## Dados mínimos esperados do ERP

O contrato definitivo será detalhado campo por campo, mas a entrada mínima deverá representar os fatos comerciais necessários para processar a operação.

### Contexto

- `company_id`;
- `document_type`;
- `operation`;
- `external_reference`;
- `idempotency_key`;
- `integration_id`, quando a entrada usar uma integração configurada.

O `group_id` e o nome do schema não fazem parte do payload. Eles são obtidos internamente a partir da API Key.

### Destinatário

- documento de identificação;
- nome ou razão social;
- endereço e demais informações exigidas para o cenário;
- indicadores fiscais necessários quando aplicáveis.

O detalhamento da obrigatoriedade dependerá do tipo de operação e das regras vigentes.

### Itens

- código ou identificação do produto;
- descrição quando não puder ser obtida do cadastro;
- quantidade;
- valor unitário;
- descontos e acréscimos comerciais quando existirem.

NCM, unidade, origem e classificações tributárias poderão ser completados por cadastros e regras somente quando houver uma fonte confiável e uma decisão determinística.

### Informações comerciais complementares

- frete;
- transporte;
- pagamento;
- referências fiscais;
- informações adicionais.

Esses blocos serão obrigatórios apenas quando o cenário fiscal ou comercial exigir.

## Matriz de preenchimento

Cada campo do contrato `nfe/v1` deverá possuir uma definição com, no mínimo:

| Propriedade | Pergunta respondida |
| --- | --- |
| Caminho | Onde o campo existe no contrato interno? |
| Descrição | Qual é o significado fiscal e comercial? |
| Tipo e formato | Como o valor é representado após a normalização? |
| Obrigatoriedade | É obrigatório sempre ou apenas em determinadas condições? |
| Origem principal | ERP, cadastro, regra fiscal ou motor fiscal? |
| Origem alternativa | Existe uma fonte secundária permitida? |
| Precedência | Qual fonte prevalece em caso de conflito? |
| Sobrescrita | O ERP pode substituir o cadastro ou a regra? |
| Enriquecimento | A plataforma pode completar o campo? |
| Validação | Quais regras precisam ser aplicadas? |
| Ausência | O documento fica pendente, inválido ou o bloco é opcional? |
| Rastreabilidade | Qual origem e versão devem ser registradas? |
| Erro | Qual código e mensagem serão retornados ao cliente? |

Exemplo inicial da matriz:

| Campo | Origem principal | ERP pode enviar? | Plataforma pode completar? | Regra inicial |
| --- | --- | --- | --- | --- |
| `company_id` | ERP | Sim | Não | Deve existir no schema do grupo autenticado. |
| Dados do emitente | Cadastro da empresa | Não | Sim | São obtidos pela empresa selecionada. |
| Quantidade | ERP | Sim | Não | Deve representar a operação comercial. |
| Valor unitário | ERP | Sim | Não | Deve representar a operação comercial. |
| Descrição do produto | ERP ou cadastro | Sim | Sim | Pode ser obtida pelo código do produto. |
| NCM | Cadastro do produto | Sim | Sim | Qualquer valor recebido deve ser validado. |
| CFOP | Regra fiscal | Sim | Sim | Sugestão do ERP não dispensa validação. |
| CST ou CSOSN | Regra fiscal | Sim | Sim | Depende do regime e do cenário tributário. |
| Série e número | Configuração da empresa | Não | Sim | Serão controlados pela plataforma. |
| Bases e tributos | Motor fiscal | Parcialmente | Sim | O resultado oficial é calculado ou validado pela plataforma. |

Esta tabela é apenas a direção inicial. A matriz definitiva deverá detalhar todos os campos aplicáveis ao cenário do MVP.

## Precedência e conflitos

Não haverá uma única ordem de precedência válida para todos os campos.

- Fatos comerciais, como quantidade e valor unitário, têm o ERP como fonte principal.
- Dados do emitente têm o cadastro da empresa como fonte oficial.
- Dados estáveis do produto podem vir do cadastro, com política explícita para aceitar ou rejeitar valores enviados.
- Decisões tributárias pertencem às regras fiscais.
- Cálculos e totais oficiais pertencem ao motor fiscal.

Quando duas fontes apresentarem valores diferentes, a plataforma deverá seguir a política específica do campo. O conflito deverá ser validado e registrado; não poderá ser resolvido silenciosamente.

## Ausência e ambiguidade

A plataforma somente completará um campo quando existir uma fonte confiável e uma regra determinística.

Se um dado fiscal obrigatório estiver ausente, inválido ou tiver mais de uma interpretação possível, o documento não será transmitido. Ele deverá ficar com uma pendência ou erro claro, indicando:

- campo afetado;
- motivo da rejeição interna;
- dado recebido, quando seguro exibi-lo;
- informação necessária para correção;
- origem ou regra que não pôde ser determinada.

A plataforma não enviará um documento sabidamente incompleto para descobrir se a SEFAZ o aceitará.

## Rastreabilidade

O documento fiscal deverá registrar a origem dos campos relevantes e as versões das regras usadas no processamento.

Exemplo:

```text
Quantidade: payload do ERP
NCM: cadastro do produto
CFOP: regra fiscal versão 1.0
Regime tributário: cadastro da empresa
ICMS: motor fiscal versão 1.0
```

Além da origem, deverão ser preservados:

- payload original;
- resultado do mapeamento;
- documento normalizado;
- valores enriquecidos;
- conflitos encontrados;
- versão do contrato `nfe/v1`;
- versão do módulo fiscal;
- versão do pacote de regras;
- data e hora da decisão fiscal.

## Experiência do cliente

A API será amigável porque o cliente não precisará repetir informações que já estejam cadastradas nem reproduzir internamente todo o conhecimento fiscal da plataforma.

A flexibilidade será oferecida por meio de:

- contrato padrão bem documentado;
- mapeamento de payloads customizados;
- normalização de formatos;
- enriquecimento por cadastros confiáveis;
- mensagens de erro objetivas;
- indicação clara de campos pendentes;
- rastreabilidade das decisões tomadas.

Essa facilidade não permitirá que o cliente desative validações fiscais ou force valores incompatíveis com a operação.

## Conformidade legal e técnica

As regras definitivas deverão ser baseadas nas fontes oficiais aplicáveis ao cenário, incluindo:

- legislação vigente;
- Ajustes SINIEF e demais atos aplicáveis;
- Manual de Orientação do Contribuinte;
- schemas XML oficiais;
- Notas Técnicas e Informes Técnicos vigentes;
- regras de validação da NF-e;
- particularidades da UF da empresa emitente.

Essas referências deverão ser versionadas e acompanhadas continuamente. Uma alteração oficial poderá modificar campos, validações, tabelas ou regras de preenchimento sem alterar o contrato comercial de entrada imediatamente.

Mudanças fiscais deverão passar por análise, implementação, testes em homologação e ativação controlada. Documentos já processados continuarão associados às versões usadas na decisão original.

## Ordem de conclusão desta etapa

1. Confirmar empresa, UF, regime tributário e operação real do MVP.
2. Fechar os dados mínimos que o ERP deve enviar.
3. Definir os cadastros mínimos de empresa, produto e destinatário.
4. Detalhar a matriz completa do `nfe/v1` para o cenário escolhido.
5. Definir precedência, sobrescrita e enriquecimento de cada campo.
6. Definir erros para ausência, valor inválido e conflito entre fontes.
7. Definir o registro de origem e versão das regras.
8. Validar a matriz contra a documentação oficial vigente e o cenário fiscal real.

## Estado deste planejamento

A direção arquitetural está definida: entrada flexível, responsabilidade comercial do ERP, enriquecimento por cadastros, decisão por regras fiscais e cálculos pelo motor fiscal.

A etapa somente será considerada concluída depois que a empresa real do MVP for definida e a matriz de campos do `nfe/v1` estiver completa e validada.
