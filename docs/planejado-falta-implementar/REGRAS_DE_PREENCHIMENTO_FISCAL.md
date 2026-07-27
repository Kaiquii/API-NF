# Regras de preenchimento fiscal

> Status: planejamento concluído, aguardando implementação.

## Objetivo

Definir de onde vem cada informação necessária para formar um documento fiscal, quem é responsável pelo dado, quando a plataforma pode completá-lo e quais validações devem ocorrer antes da emissão.

Esta etapa começa pela NF-e como primeiro módulo fiscal, mas sua estrutura não será baseada em uma empresa ou segmento específico. O detalhamento campo a campo está consolidado em `MATRIZ_DE_PREENCHIMENTO_NFE_V1.md`.

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

## Cobertura progressiva e independente de cliente

A estrutura fiscal da plataforma será definida por documento e cenário, nunca por uma empresa específica.

```text
Contrato base do documento
  -> campos condicionais
  -> dimensões do cenário fiscal
  -> pacote de regras vigente
  -> decisão fiscal
```

A NF-e modelo 55 será o primeiro módulo planejado. A matriz base deverá representar os conceitos comuns do documento sem assumir previamente uma UF, um regime tributário ou um segmento comercial.

Empresas reais poderão ser usadas como casos de validação e homologação. Elas não definirão os nomes, a estrutura ou as regras gerais do contrato.

A arquitetura será preparada para vários segmentos desde o início, mas a disponibilidade será progressiva. Um cenário somente poderá ser declarado como suportado depois que suas regras estiverem documentadas, implementadas, testadas e homologadas.

Operações ainda não suportadas deverão ser recusadas de forma explícita. A plataforma não poderá aplicar uma regra aproximada ou reutilizar silenciosamente um cenário incompatível.

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

### Dados fiscais enviados pelo ERP

NCM, CEST, CFOP, CST, CSOSN, alíquotas, bases e valores de tributos não serão obrigatórios no contrato padrão apenas porque o ERP os possui.

Quando forem enviados, deverão ser tratados como informação de origem ou sugestão para conferência. A plataforma continuará responsável por validar o enquadramento e determinar o resultado fiscal oficial por meio dos cadastros, das regras e do motor fiscal.

## Cadastros mínimos

### Empresa

O cadastro da empresa deverá possuir, com vigência quando aplicável:

- CNPJ;
- razão social e nome fantasia;
- inscrição estadual e municipal, quando aplicável;
- endereço, município, código de município e UF;
- regime tributário;
- CNAE e configurações fiscais necessárias;
- documentos fiscais habilitados;
- séries disponíveis por modelo e ambiente;
- benefícios ou regimes especiais;
- status e período de validade das configurações.

Dados oficiais do emitente não poderão ser sobrescritos pelo payload da operação.

### Produto

O cadastro fiscal do produto deverá permitir:

- identificar o produto pelo código ou SKU;
- armazenar descrição;
- NCM;
- CEST quando aplicável;
- origem da mercadoria;
- unidade comercial e unidade tributável;
- fator de conversão entre unidades;
- GTIN quando existir;
- características tributárias especiais;
- vigência do cadastro fiscal.

Quantidade, valor unitário, desconto e demais fatos da venda continuam pertencendo ao ERP.

### Destinatário

O cadastro do destinatário poderá armazenar:

- CPF, CNPJ ou identificação estrangeira;
- nome ou razão social;
- indicador de contribuinte;
- inscrição estadual quando aplicável;
- endereço completo;
- SUFRAMA e outras inscrições condicionais;
- informações de contato.

O cadastro poderá completar dados recorrentes, mas não poderá trocar silenciosamente a identidade do destinatário indicada pelo ERP.

CPF, CNPJ, inscrições e demais documentos serão tratados como texto. O modelo não assumirá que identificadores fiscais são exclusivamente numéricos.

## Matrizes de preenchimento

O planejamento será dividido entre uma matriz base e regras condicionais.

### Matriz base

A matriz base definirá os campos e blocos comuns da NF-e, independentemente do segmento da empresa:

- contexto do documento;
- identificação;
- emitente;
- destinatário e pessoas autorizadas;
- itens;
- tributos por item;
- totais;
- transporte;
- cobrança;
- pagamentos;
- referências fiscais;
- informações adicionais e processos.

Ela determinará o significado interno dos dados, seus tipos, fontes possíveis e regras gerais de normalização.

### Regras condicionais

As regras condicionais definirão quando campos, grupos fiscais ou validações adicionais serão aplicáveis.

Exemplos:

```text
se a operação for interestadual -> avaliar regras entre UFs
se o destinatário for contribuinte -> aplicar validações correspondentes
se o produto estiver sujeito a ST -> exigir e calcular o grupo aplicável
se houver benefício fiscal -> exigir código e fundamentação
se houver frete -> exigir os dados compatíveis com a modalidade
```

Condições pertencem ao cenário fiscal, e não ao cliente. Uma regra aplicável poderá ser reutilizada por qualquer grupo que apresente as mesmas características fiscais.

### Definição de cada campo

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

Esta tabela é apenas a direção inicial. A matriz definitiva deverá detalhar os campos base e identificar as condições que habilitam os demais campos.

## Dimensões dos cenários fiscais

Os pacotes de regras deverão tomar decisões a partir de dimensões explícitas, como:

```text
tipo e versão do documento
vigência da regra
regime tributário da empresa
UF e município de origem
UF e município de destino
tipo e finalidade da operação
perfil fiscal do destinatário
classificação e origem do produto
características tributárias do item
benefícios ou regimes especiais
```

Essas dimensões serão obtidas dos fatos comerciais, dos cadastros e do contexto fiscal. O nome do grupo, o segmento comercial informado pelo cliente ou uma configuração livre não poderão substituir as condições fiscais objetivas.

## Pacotes de regras e habilitação

Regras fiscais serão organizadas em pacotes versionados, imutáveis e com vigência definida.

Cada cenário suportado deverá possuir um manifesto contendo:

```text
scenario_id
document_type
contract_version
operation_types
purposes
tax_regimes
origin_scope
destination_scope
recipient_profiles
product_constraints
special_groups
effective_from
effective_until
rule_package
xml_schema_package
status
```

Status do manifesto:

```text
planned
implemented
homologated
active
suspended
retired
```

Somente um cenário `active` poderá autorizar emissão em produção. Homologação e produção terão habilitações independentes.

Um cenário somente poderá ser declarado como suportado quando:

1. todas as dimensões de entrada estiverem definidas;
2. todos os campos aplicáveis tiverem origem e precedência;
3. não houver regra ambígua;
4. cálculos e arredondamentos estiverem especificados no pacote fiscal;
5. o XML validar contra o schema oficial aplicável;
6. casos positivos e negativos estiverem automatizados;
7. a emissão estiver validada em homologação;
8. versões, vigências, fontes e limitações estiverem registradas.

Uma empresa poderá usar determinado cenário quando seus dados e sua operação forem compatíveis com essa cobertura. A habilitação não exigirá uma versão própria da API ou código exclusivo por cliente.

## Limite entre a matriz e os pacotes fiscais

A matriz do `nfe/v1` define, campo por campo:

- significado e tipo do dado;
- origem principal e origens alternativas permitidas;
- obrigatoriedade base ou condicional;
- precedência em caso de conflito;
- possibilidade de sobrescrita e enriquecimento;
- comportamento quando o dado estiver ausente ou inválido;
- rastreabilidade e erro semântico.

As fórmulas tributárias e decisões legais específicas não serão incorporadas à matriz base. Elas pertencem aos pacotes fiscais versionados, que deverão considerar as dimensões do cenário e declarar vigência, fontes oficiais, cálculos, arredondamentos, exceções e casos de teste.

```text
Matriz do nfe/v1
  -> define quais informações existem e quem responde por elas

Pacote fiscal versionado
  -> define como enquadrar e calcular para um cenário vigente
```

Essa separação permite manter o contrato interno estável quando uma regra tributária mudar. Uma atualização legal poderá gerar uma nova versão do pacote fiscal sem obrigar todos os clientes a alterar imediatamente o formato comercial enviado à API.

## Precedência e conflitos

Não haverá uma única ordem de precedência válida para todos os campos.

| Categoria | Autoridade principal | Tratamento |
| --- | --- | --- |
| Identidade do grupo | API Key | O payload não pode alterar. |
| Ambiente | API Key e configuração | O payload não pode alternar homologação e produção. |
| Empresa emitente | `company_id` e cadastro | Dados oficiais não podem ser sobrescritos pelo ERP. |
| Fatos comerciais | ERP | Cadastro não pode inventar quantidade, valor ou intenção da operação. |
| Dados fiscais estáveis do produto | Cadastro | O valor do ERP depende da política específica do campo. |
| Identidade do destinatário | ERP | Cadastro pode completar, mas não substituir silenciosamente. |
| Enquadramento fiscal | Pacote de regras | Sugestões do ERP precisam ser validadas. |
| Cálculos e totais | Motor fiscal | Valores do ERP servem apenas para conferência. |
| Número, série e chave de acesso | Plataforma | Não podem ser informados livremente pelo ERP. |
| Textos fiscais obrigatórios | Pacote de regras | ERP não pode remover ou sobrescrever. |

Quando duas fontes apresentarem valores diferentes, a plataforma deverá seguir a política específica do campo. O conflito deverá ser validado e registrado; não poderá ser resolvido silenciosamente.

## Política de enriquecimento

Um campo somente poderá ser completado automaticamente quando:

1. a política do campo permitir;
2. existir uma fonte ativa e identificável;
3. houver somente uma decisão válida para o contexto;
4. a regra aplicada estiver vigente;
5. a transformação puder ser auditada.

Se essas condições não forem atendidas, o documento permanecerá pendente ou inválido.

## Ausência e ambiguidade

A plataforma somente completará um campo quando existir uma fonte confiável e uma regra determinística.

Se um dado fiscal obrigatório estiver ausente, inválido ou tiver mais de uma interpretação possível, o documento não será transmitido. Ele deverá ficar com uma pendência ou erro claro, indicando:

- campo afetado;
- motivo da rejeição interna;
- dado recebido, quando seguro exibi-lo;
- informação necessária para correção;
- origem ou regra que não pôde ser determinada.

A plataforma não enviará um documento sabidamente incompleto para descobrir se a SEFAZ o aceitará.

O tratamento será separado por natureza:

| Situação | Comportamento |
| --- | --- |
| Dado ausente e corrigível | Documento fica com `pending_data`. |
| Dado inválido | Validação interna bloqueia a emissão. |
| Conflito entre fontes | Aplicar política explícita ou solicitar correção. |
| Regra não encontrada | Tratar como configuração fiscal incompleta. |
| Regras ambíguas | Bloquear como erro de configuração. |
| Cenário não suportado | Informar explicitamente que a cobertura ainda não está disponível. |
| Rejeição da autoridade fiscal | Registrar somente depois de uma transmissão efetiva. |

Validação interna e rejeição da autoridade fiscal são conceitos diferentes e não deverão compartilhar a mesma causa ou mensagem.

### Erros semânticos

| Código | Significado |
| --- | --- |
| `FISCAL_FIELD_REQUIRED` | Campo obrigatório ausente para o cenário. |
| `FISCAL_FIELD_INVALID` | Tipo, formato, tamanho ou domínio inválido. |
| `FISCAL_FIELD_CONFLICT` | Fontes autorizadas apresentaram valores incompatíveis. |
| `FISCAL_REFERENCE_NOT_FOUND` | Cadastro ou documento referenciado não existe. |
| `FISCAL_PRODUCT_DATA_MISSING` | Cadastro fiscal do produto é insuficiente. |
| `FISCAL_RULE_NOT_FOUND` | Nenhuma regra vigente atende ao cenário. |
| `FISCAL_RULE_AMBIGUOUS` | Mais de uma regra incompatível foi encontrada. |
| `FISCAL_SCENARIO_UNSUPPORTED` | Cenário ainda não foi habilitado. |
| `FISCAL_CALCULATION_MISMATCH` | Valor de conferência diverge do cálculo oficial. |
| `FISCAL_DOCUMENT_NOT_READY` | Existem pendências bloqueantes. |

A Etapa 6 definirá status HTTP, envelope público e formato da resposta. Nesta etapa são definidos apenas o significado e a causa fiscal dos erros.

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

Para cada campo relevante, a rastreabilidade deverá permitir registrar:

```text
field_path
source_value
normalized_value
resolved_value
source_type
source_reference
rule_package
rule_identifier
rule_version
effective_at
decided_at
```

Além da origem, deverão ser preservados:

- payload original;
- resultado do mapeamento;
- documento normalizado;
- snapshot enriquecido;
- conflitos encontrados;
- versão do contrato `nfe/v1`;
- versão do módulo fiscal;
- versão do pacote de regras;
- pacote de schema XML aplicável;
- data e hora da decisão fiscal.

Depois da autorização, o snapshot fiscal não poderá ser reinterpretado por versões novas de cadastros ou regras.

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

1. Definir a matriz base do `nfe/v1`, independente de segmento.
2. Fechar os fatos comerciais mínimos que o ERP deve enviar.
3. Definir os cadastros mínimos de empresa, produto e destinatário.
4. Identificar as dimensões que determinam os cenários fiscais.
5. Detalhar as regras condicionais dos campos e grupos fiscais.
6. Definir precedência, sobrescrita e enriquecimento de cada campo.
7. Definir erros para ausência, valor inválido, conflito e cenário não suportado.
8. Definir o registro de origem, vigência e versão das regras.
9. Definir os critérios para habilitar e declarar um cenário como suportado.
10. Documentar nos pacotes fiscais, e não na matriz base, as fórmulas e decisões legais específicas.
11. Validar as matrizes contra a documentação oficial vigente e casos reais de diferentes segmentos.

## Estado deste planejamento

A Etapa 4 está concluída para planejamento. Foram definidos a matriz base do `nfe/v1`, os responsáveis por cada categoria de dado, as regras condicionais, a precedência entre fontes, o enriquecimento permitido, os erros semânticos, a rastreabilidade, o manifesto de cobertura, os critérios de ativação e o catálogo mínimo de testes.

Casos de empresas reais serão usados durante a implementação e a homologação para confirmar a abrangência do modelo. Nenhuma empresa será usada como padrão estrutural da plataforma.

As fórmulas e decisões legais específicas continuarão nos pacotes fiscais versionados. Numeração, idempotência, contratos HTTP, integração com a SEFAZ, segurança de certificados e modelo físico do banco permanecem reservados às próximas etapas.
