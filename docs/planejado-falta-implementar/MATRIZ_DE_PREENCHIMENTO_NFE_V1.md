# Matriz de preenchimento do contrato nfe/v1

> Status: planejamento concluído, aguardando implementação.

## Objetivo

Detalhar os campos semânticos do contrato interno `nfe/v1`, suas origens, obrigatoriedade, precedência e comportamento quando ausentes.

Este documento não replica o XML da NF-e. O contrato interno usa nomes estáveis e compreensíveis; o módulo fiscal será responsável por convertê-los para o schema XML oficial vigente na data da emissão.

## Escopo

```text
document_type: nfe
document_version: 1
fiscal_model: 55
```

A matriz é independente de cliente, segmento, UF e regime tributário. Campos condicionais são habilitados pelo cenário fiscal e pelo pacote de regras aplicável.

## Referências oficiais

Base verificada em 27 de julho de 2026:

- [MOC 7.0 - Anexo I - Leiaute e Regras de Validação da NF-e/NFC-e](https://www.nfe.fazenda.gov.br/portal/exibirArquivo.aspx?conteudo=J+I+v4eN00E%3D);
- [Schemas XML oficiais da NF-e](https://www.nfe.fazenda.gov.br/portal/listaConteudo.aspx?AspxAutoDetectCookieSupport=1&tipoConteudo=BMPFMBoln3w%3D);
- [Notas Técnicas vigentes da NF-e](https://www.nfe.fazenda.gov.br/portal/listaConteudo.aspx?tipoConteudo=6WfrpZYE4Ik%3D);
- Nota Técnica 2025.002 v.1.50, referente à Reforma Tributária do Consumo;
- Nota Técnica 2026.004 v.1.01, referente à adequação para CNPJ alfanumérico.

Essas versões não ficam fixadas para sempre no contrato. Cada pacote fiscal deverá declarar as fontes, versões e vigências efetivamente utilizadas.

## Legenda

### Origem

| Código | Origem |
| --- | --- |
| `ERP` | Payload padrão ou integração mapeada. |
| `CAD` | Cadastro de empresa, produto, destinatário ou terceiro. |
| `REGRA` | Pacote fiscal versionado. |
| `MOTOR` | Motor de cálculo fiscal. |
| `SISTEMA` | Plataforma, autenticação ou processamento interno. |

### Obrigatoriedade

| Código | Significado |
| --- | --- |
| `B` | Campo base necessário para o contrato. |
| `C` | Campo obrigatório quando a condição ocorrer. |
| `A` | Campo obtido ou calculado automaticamente. |
| `I` | Campo exclusivamente interno. |

Um campo `C` não é opcional quando sua condição estiver presente.

## Regras gerais

- O grupo é obtido pela API Key e não aparece no payload.
- O ambiente vem da credencial e da configuração, não de uma escolha livre do ERP.
- O emitente é localizado por `company_id` dentro do schema do grupo.
- CPF, CNPJ, inscrições, chaves e códigos que não participam de cálculos são texto.
- Quantidades, valores, destinatário e intenção comercial não podem ser inventados.
- Dados fiscais enviados pelo ERP são fatos ou sugestões conforme a política do campo.
- Enquadramento fiscal pertence ao pacote de regras.
- Cálculos e totais oficiais pertencem ao motor fiscal.
- Numeração e chave de acesso pertencem à plataforma.
- Campo condicional ausente bloqueia a emissão quando sua condição estiver ativa.
- Cenário sem pacote fiscal ativo retorna `FISCAL_SCENARIO_UNSUPPORTED`.

## Envelope

| Campo | Obrig. | Origem | ERP informa? | Regra |
| --- | --- | --- | --- | --- |
| `id` | `I` | `SISTEMA` | Não | UUID interno e imutável. |
| `document_type` | `B` | Rota/integração | Sim | Deve ser `nfe`. |
| `document_version` | `I` | `SISTEMA` | Não | Versão `1`. |
| `company_id` | `B` | `ERP` | Sim | Empresa ativa pertencente ao grupo autenticado. |
| `integration_id` | `C` | Rota/integração | Parcial | Obrigatório na entrada configurada. |
| `environment` | `A` | `SISTEMA` | Não | Homologação ou produção conforme credencial e empresa. |
| `operation` | `B` | `ERP` | Sim | Intenção comercial normalizada. |
| `external_reference` | `B` | `ERP` | Sim | Identificador no sistema de origem. |
| `idempotency_key` | `B` | Header/ERP | Sim | Semântica transacional será detalhada na Etapa 5. |
| `data` | `B` | Payload normalizado | Sim | Conteúdo específico do `nfe/v1`. |

## Identificação

| Campo | Obrig. | Origem | ERP informa? | Regra ou condição |
| --- | --- | --- | --- | --- |
| `identification.fiscal_model` | `A` | Contrato | Não | Modelo 55. |
| `identification.layout_version` | `I` | Módulo fiscal | Não | Pacote físico de schema XML usado. |
| `identification.access_key` | `I` | `SISTEMA` | Não | Gerada após reserva de numeração. |
| `identification.numeric_code` | `I` | `SISTEMA` | Não | Componente da chave de acesso. |
| `identification.operation_nature` | `B` | `ERP`/`REGRA` | Sim | Descrição da natureza validada para a operação. |
| `identification.direction` | `A` | `REGRA` | Como fato | Entrada ou saída. |
| `identification.destination_scope` | `A` | `REGRA` | Não | Interna, interestadual ou exterior. |
| `identification.series` | `A` | Configuração | Não | Série da empresa, modelo e ambiente. |
| `identification.number` | `I` | `SISTEMA` | Não | Reservado pela plataforma. |
| `identification.issued_at` | `A` | `SISTEMA`/`ERP` | Condicional | Política de horário será validada. |
| `identification.exit_entry_at` | `C` | `ERP` | Sim | Quando aplicável à circulação ou entrada. |
| `identification.purpose` | `B` | `ERP`/`REGRA` | Sim | Normal, complementar, ajuste ou devolução quando suportado. |
| `identification.final_consumer` | `B` | `ERP`/`REGRA` | Sim | Coerente com destinatário e operação. |
| `identification.presence_indicator` | `C` | `ERP` | Sim | Conforme forma da operação. |
| `identification.intermediary_indicator` | `C` | `ERP` | Sim | Quando houver intermediador. |
| `identification.print_layout` | `A` | Configuração | Não | Compatível com documento e cenário. |
| `identification.emission_type` | `A` | `SISTEMA` | Não | Normal ou contingência habilitada. |
| `identification.contingency_at` | `C` | `SISTEMA` | Não | Somente em contingência. |
| `identification.contingency_reason` | `C` | `SISTEMA` | Não | Somente em contingência. |
| `identification.issuer_municipality_code` | `A` | `CAD` | Não | Código oficial do município do emitente. |
| `identification.application_version` | `I` | `SISTEMA` | Não | Versão do módulo emissor. |

## Emitente

O ERP seleciona a empresa, mas não sobrescreve seus dados oficiais.

| Campo | Obrig. | Origem | Regra |
| --- | --- | --- | --- |
| `issuer.company_id` | `A` | Envelope | Referência da empresa selecionada. |
| `issuer.document` | `A` | `CAD` | CNPJ textual conforme padrão vigente. |
| `issuer.legal_name` | `A` | `CAD` | Razão social oficial. |
| `issuer.trade_name` | `C` | `CAD` | Quando cadastrado e aplicável. |
| `issuer.state_registration` | `A` | `CAD` | Inscrição estadual vigente. |
| `issuer.state_registration_substitute` | `C` | `CAD` | Quando aplicável. |
| `issuer.municipal_registration` | `C` | `CAD` | Quando exigida. |
| `issuer.cnae` | `C` | `CAD` | Quando exigido pelo cenário. |
| `issuer.tax_regime` | `A` | `CAD` | Regime vigente na data da operação. |
| `issuer.address.street` | `A` | `CAD` | Logradouro oficial. |
| `issuer.address.number` | `A` | `CAD` | Número. |
| `issuer.address.complement` | `C` | `CAD` | Complemento. |
| `issuer.address.district` | `A` | `CAD` | Bairro. |
| `issuer.address.city_code` | `A` | `CAD` | Código oficial do município. |
| `issuer.address.city_name` | `A` | `CAD` | Nome coerente com o código. |
| `issuer.address.state` | `A` | `CAD` | UF. |
| `issuer.address.postal_code` | `A` | `CAD` | CEP normalizado. |
| `issuer.address.country_code` | `A` | `CAD` | Código do país. |
| `issuer.address.country_name` | `A` | `CAD` | Nome coerente com o código. |
| `issuer.address.phone` | `C` | `CAD` | Quando informado ou exigido. |

## Destinatário

O cadastro pode completar dados recorrentes, mas não pode trocar silenciosamente a identidade indicada pelo ERP.

| Campo | Obrig. | Origem | ERP informa? | Regra ou condição |
| --- | --- | --- | --- | --- |
| `recipient.person_type` | `B` | `ERP`/documento | Sim | Pessoa física, jurídica ou estrangeira. |
| `recipient.document` | `C` | `ERP` | Sim | CPF ou CNPJ textual conforme cenário. |
| `recipient.foreign_id` | `C` | `ERP` | Sim | Identificação estrangeira quando aplicável. |
| `recipient.name` | `B` | `ERP`/`CAD` | Sim | Identidade da operação. |
| `recipient.state_registration_indicator` | `B` | `ERP`/`REGRA` | Sim | Coerente com perfil fiscal. |
| `recipient.state_registration` | `C` | `ERP`/`CAD` | Sim | Quando o indicador exigir. |
| `recipient.suframa_registration` | `C` | `ERP`/`CAD` | Sim | Quando aplicável. |
| `recipient.municipal_registration` | `C` | `ERP`/`CAD` | Sim | Quando aplicável. |
| `recipient.email` | `C` | `ERP`/`CAD` | Sim | Normalizado. |
| `recipient.address.street` | `C` | `ERP`/`CAD` | Sim | Conforme obrigatoriedade do cenário. |
| `recipient.address.number` | `C` | `ERP`/`CAD` | Sim | Conforme obrigatoriedade do cenário. |
| `recipient.address.complement` | `C` | `ERP`/`CAD` | Sim | Quando houver. |
| `recipient.address.district` | `C` | `ERP`/`CAD` | Sim | Conforme cenário. |
| `recipient.address.city_code` | `C` | `ERP`/`CAD` | Sim | Código oficial; exterior segue regra própria. |
| `recipient.address.city_name` | `C` | `ERP`/`CAD` | Sim | Coerente com o código. |
| `recipient.address.state` | `C` | `ERP`/`CAD` | Sim | Determina dimensões fiscais. |
| `recipient.address.postal_code` | `C` | `ERP`/`CAD` | Sim | Normalizado conforme país. |
| `recipient.address.country_code` | `C` | `ERP`/`CAD` | Sim | Determina cenário nacional ou exterior. |
| `recipient.address.country_name` | `C` | `ERP`/`CAD` | Sim | Coerente com o código. |
| `recipient.address.phone` | `C` | `ERP`/`CAD` | Sim | Quando informado ou exigido. |

## Pessoas autorizadas a acessar o XML

| Campo | Obrig. | Origem | Regra |
| --- | --- | --- | --- |
| `authorized_xml[].document` | `C` | `ERP`/`CAD` | CPF ou CNPJ autorizado, limitado pelo leiaute vigente. |

## Itens

| Campo | Obrig. | Origem | ERP informa? | Regra ou condição |
| --- | --- | --- | --- | --- |
| `items[].line_number` | `A` | `SISTEMA` | Não | Sequencial e único. |
| `items[].product_code` | `B` | `ERP` | Sim | Identificador no sistema de origem. |
| `items[].gtin` | `C` | `ERP`/`CAD` | Sim | Validado conforme regras vigentes. |
| `items[].description` | `B` | `ERP`/`CAD` | Sim | Deve representar o item vendido. |
| `items[].ncm` | `B` | `CAD`/`ERP` | Sim | Código vigente e validado. |
| `items[].nve` | `C` | `CAD`/`ERP` | Sim | Quando aplicável. |
| `items[].extipi` | `C` | `CAD`/`ERP` | Sim | Quando aplicável. |
| `items[].cest` | `C` | `CAD`/`REGRA` | Referência | Quando o cenário exigir. |
| `items[].tax_benefit_code` | `C` | `REGRA`/`CAD` | Referência | Quando houver benefício aplicável. |
| `items[].scale_indicator` | `C` | `CAD`/`REGRA` | Sim | Quando exigido. |
| `items[].manufacturer_document` | `C` | `CAD`/`ERP` | Sim | Condicionado ao indicador de escala. |
| `items[].cfop` | `A` | `REGRA` | Sugestão | Resultado final pertence ao pacote fiscal. |
| `items[].commercial_unit` | `B` | `ERP`/`CAD` | Sim | Unidade da operação. |
| `items[].commercial_quantity` | `B` | `ERP` | Sim | Fato comercial. |
| `items[].commercial_unit_amount` | `B` | `ERP` | Sim | Fato comercial. |
| `items[].gross_amount` | `A` | `MOTOR` | Conferência | Calculado com precisão definida. |
| `items[].taxable_gtin` | `C` | `CAD`/`ERP` | Sim | Quando aplicável. |
| `items[].taxable_unit` | `A` | `CAD`/`REGRA` | Fato | Pode exigir conversão. |
| `items[].taxable_quantity` | `A` | `MOTOR`/`CAD` | Fato | Derivada pela conversão. |
| `items[].taxable_unit_amount` | `A` | `MOTOR` | Conferência | Coerente com a conversão. |
| `items[].freight_amount` | `C` | `MOTOR` | Total | Rateio determinístico. |
| `items[].insurance_amount` | `C` | `MOTOR` | Total | Rateio determinístico. |
| `items[].discount_amount` | `C` | `ERP`/`MOTOR` | Sim | Não pode invalidar o item. |
| `items[].other_amount` | `C` | `ERP`/`MOTOR` | Sim | Natureza conhecida. |
| `items[].included_in_total` | `A` | `REGRA` | Fato | Conforme composição do documento. |
| `items[].purchase_order` | `C` | `ERP` | Sim | Quando utilizado. |
| `items[].purchase_order_item` | `C` | `ERP` | Sim | Dependente do pedido. |
| `items[].additional_information` | `C` | `ERP`/`REGRA` | Sim | Texto validado. |

## Grupos especiais de item

| Grupo | Condição |
| --- | --- |
| `items[].imports[]` | Mercadoria ou operação importada. |
| `items[].exports[]` | Exportação com detalhamento do item. |
| `items[].traceability[]` | Lote, fabricação, validade ou agregação exigidos. |
| `items[].medicine` | Medicamento ou produto sujeito ao grupo específico. |
| `items[].fuel` | Combustível ou derivado sujeito ao grupo específico. |
| `items[].vehicle` | Veículo novo. |
| `items[].weapons[]` | Produto sujeito ao grupo de armas. |
| `items[].regulated_product` | Outro subcontrato oficial habilitado e versionado. |

Cada grupo especial terá contrato próprio antes de ser declarado suportado. Não será usado JSON livre para esconder campos ainda não modelados.

## Tributos por item

| Grupo | Origem | Condição |
| --- | --- | --- |
| `items[].taxes.icms` | `REGRA` + `MOTOR` | Conforme regime, operação, produto e destino. |
| `items[].taxes.ipi` | `REGRA` + `MOTOR` | Quando houver incidência ou informação obrigatória. |
| `items[].taxes.import_tax` | `REGRA` + `MOTOR` | Importação suportada. |
| `items[].taxes.pis` | `REGRA` + `MOTOR` | Conforme enquadramento vigente. |
| `items[].taxes.cofins` | `REGRA` + `MOTOR` | Conforme enquadramento vigente. |
| `items[].taxes.issqn` | `REGRA` + `MOTOR` | Cenário suportado com serviço no documento. |
| `items[].taxes.ibs_cbs` | `REGRA` + `MOTOR` | Conforme vigência da Reforma Tributária. |
| `items[].taxes.selective_tax` | `REGRA` + `MOTOR` | Quando previsto e vigente. |
| `items[].taxes.estimated_total` | `REGRA` + `MOTOR` | Quando exigido. |

Cada grupo tributário deverá registrar enquadramento, códigos, bases, alíquotas, reduções, diferimentos, créditos, valores, parâmetros, arredondamentos e pacote fiscal usado.

As fórmulas não ficam nesta matriz. Elas pertencem ao pacote fiscal versionado.

## Totais

| Campo ou grupo | Obrig. | Origem | Regra |
| --- | --- | --- | --- |
| `totals.products` | `A` | `MOTOR` | Soma dos produtos. |
| `totals.freight` | `C` | `MOTOR` | Soma coerente com os rateios. |
| `totals.insurance` | `C` | `MOTOR` | Soma coerente com os rateios. |
| `totals.discount` | `C` | `MOTOR` | Soma dos descontos. |
| `totals.other` | `C` | `MOTOR` | Soma dos acréscimos. |
| `totals.document` | `A` | `MOTOR` | Total final conforme composição fiscal. |
| `totals.icms` | `C` | `MOTOR` | Bases e valores aplicáveis. |
| `totals.ipi` | `C` | `MOTOR` | Quando aplicável. |
| `totals.import_tax` | `C` | `MOTOR` | Quando aplicável. |
| `totals.pis` | `C` | `MOTOR` | Quando aplicável. |
| `totals.cofins` | `C` | `MOTOR` | Quando aplicável. |
| `totals.issqn` | `C` | `MOTOR` | Quando aplicável. |
| `totals.ibs_cbs` | `C` | `MOTOR` | Conforme vigência. |
| `totals.selective_tax` | `C` | `MOTOR` | Conforme vigência. |
| `totals.withholdings` | `C` | `MOTOR` | Retenções aplicáveis. |
| `totals.estimated_taxes` | `C` | `MOTOR` | Quando exigido. |

Totais enviados pelo ERP servem para conferência. Divergências além da tolerância bloqueiam a emissão.

## Transporte

| Campo ou grupo | Obrig. | Origem | ERP informa? | Regra ou condição |
| --- | --- | --- | --- | --- |
| `shipping.freight_mode` | `B` | `ERP`/`REGRA` | Sim | Modalidade coerente com a responsabilidade pelo frete. |
| `shipping.carrier.document` | `C` | `ERP`/`CAD` | Sim | Quando houver transportador identificado. |
| `shipping.carrier.name` | `C` | `ERP`/`CAD` | Sim | Quando houver transportador identificado. |
| `shipping.carrier.state_registration` | `C` | `ERP`/`CAD` | Sim | Conforme situação cadastral do transportador. |
| `shipping.carrier.address` | `C` | `ERP`/`CAD` | Sim | Quando aplicável ao leiaute. |
| `shipping.carrier.city` | `C` | `ERP`/`CAD` | Sim | Quando aplicável ao leiaute. |
| `shipping.carrier.state` | `C` | `ERP`/`CAD` | Sim | Quando aplicável ao leiaute. |
| `shipping.retained_icms` | `C` | `REGRA` + `MOTOR` | Fatos | Somente quando houver retenção aplicável ao transporte. |
| `shipping.vehicle.plate` | `C` | `ERP` | Sim | Quando o cenário exigir identificação do veículo. |
| `shipping.vehicle.state` | `C` | `ERP` | Sim | Dependente da placa. |
| `shipping.vehicle.registry` | `C` | `ERP`/`CAD` | Sim | Quando exigido. |
| `shipping.trailers[]` | `C` | `ERP` | Sim | Quando houver reboque identificado. |
| `shipping.wagon` | `C` | `ERP` | Sim | Transporte ferroviário aplicável. |
| `shipping.ferry` | `C` | `ERP` | Sim | Transporte aquaviário aplicável. |
| `shipping.volumes[]` | `C` | `ERP` | Sim | Quantidade, espécie, marca, numeração e pesos quando informados ou exigidos. |
| `shipping.volumes[].seals[]` | `C` | `ERP` | Sim | Lacres vinculados ao volume. |

O cadastro pode completar a identificação de um transportador já indicado, mas não pode criar uma contratação de frete inexistente. Pesos, volumes, veículo e modalidade são fatos logísticos e devem permanecer coerentes entre si.

## Cobrança e duplicatas

| Campo | Obrig. | Origem | Regra ou condição |
| --- | --- | --- | --- |
| `billing.invoice.number` | `C` | `ERP` | Quando houver fatura. |
| `billing.invoice.original_amount` | `C` | `ERP`/`MOTOR` | Valor original conferido contra a operação. |
| `billing.invoice.discount_amount` | `C` | `ERP`/`MOTOR` | Quando houver desconto financeiro. |
| `billing.invoice.net_amount` | `A` | `MOTOR` | Resultado coerente com valores original e desconto. |
| `billing.installments[].number` | `C` | `ERP` | Identificador único da parcela no documento. |
| `billing.installments[].due_date` | `C` | `ERP` | Data válida quando houver duplicata. |
| `billing.installments[].amount` | `C` | `ERP`/`MOTOR` | Valor positivo e coerente com a cobrança. |

Cobrança, duplicata e pagamento são conceitos distintos. A presença de parcelas não substitui o detalhamento dos pagamentos exigidos pelo cenário.

## Pagamentos

| Campo | Obrig. | Origem | Regra ou condição |
| --- | --- | --- | --- |
| `payments[].timing` | `C` | `ERP`/`REGRA` | À vista, a prazo ou outra classificação vigente. |
| `payments[].method` | `B` | `ERP` | Meio de pagamento normalizado para a tabela oficial. |
| `payments[].amount` | `B` | `ERP`/`MOTOR` | Valor conferido com o total e o troco. |
| `payments[].description` | `C` | `ERP` | Obrigatória quando o meio exigir descrição. |
| `payments[].card.integration_type` | `C` | `ERP` | Quando houver pagamento por cartão ou equivalente. |
| `payments[].card.acquirer_document` | `C` | `ERP`/`CAD` | Conforme tipo de integração e regra vigente. |
| `payments[].card.brand` | `C` | `ERP` | Quando aplicável. |
| `payments[].card.authorization_code` | `C` | `ERP` | Quando aplicável. |
| `payments[].receiver_document` | `C` | `ERP`/`CAD` | Quando exigido no pagamento. |
| `payments[].terminal_id` | `C` | `ERP`/`CAD` | Quando exigido no pagamento. |
| `payment_change` | `C` | `ERP`/`MOTOR` | Somente quando os pagamentos superarem o total e houver troco válido. |

A plataforma não infere o meio de pagamento a partir de texto livre. Os valores podem ser recalculados para conferência, mas a forma efetivamente usada é um fato comercial informado pelo ERP.

## Referências fiscais

| Campo | Obrig. | Origem | Regra ou condição |
| --- | --- | --- | --- |
| `fiscal_references[].type` | `C` | `ERP`/`REGRA` | Tipo de documento referenciado. |
| `fiscal_references[].access_key` | `C` | `ERP`/`CAD` | Para documento eletrônico com chave de acesso. |
| `fiscal_references[].state` | `C` | `ERP` | Documento legado que exigir UF. |
| `fiscal_references[].year_month` | `C` | `ERP` | Documento legado que exigir período. |
| `fiscal_references[].issuer_document` | `C` | `ERP`/`CAD` | Documento do emitente da referência. |
| `fiscal_references[].model` | `C` | `ERP` | Modelo do documento referenciado. |
| `fiscal_references[].series` | `C` | `ERP` | Série do documento referenciado. |
| `fiscal_references[].number` | `C` | `ERP` | Número do documento referenciado. |
| `fiscal_references[].producer_registration` | `C` | `ERP`/`CAD` | Nota de produtor quando aplicável. |
| `fiscal_references[].ecf_model` | `C` | `ERP` | Referência a ECF quando aplicável. |
| `fiscal_references[].ecf_number` | `C` | `ERP` | Referência a ECF quando aplicável. |
| `fiscal_references[].coo` | `C` | `ERP` | Referência a ECF quando aplicável. |

A finalidade da NF-e pode tornar uma ou mais referências obrigatórias. A plataforma validará tipo, formato, existência, pertencimento e compatibilidade da referência, sem permitir acesso a documentos de outro grupo.

## Informações adicionais e processos

| Campo | Obrig. | Origem | Regra ou condição |
| --- | --- | --- | --- |
| `additional_information.taxpayer_text` | `C` | `ERP`/`REGRA` | Informações de interesse do contribuinte. |
| `additional_information.authority_text` | `C` | `REGRA` | Informações exigidas para o fisco; não podem ser removidas pelo ERP. |
| `additional_information.fields[]` | `C` | `ERP`/`REGRA` | Pares estruturados permitidos pelo contrato. |
| `additional_information.processes[].identifier` | `C` | `ERP`/`CAD` | Identificador do processo ou ato concessório. |
| `additional_information.processes[].origin` | `C` | `ERP`/`REGRA` | Origem normalizada conforme domínio oficial. |

Textos gerados por regras fiscais devem registrar a regra e a vigência que os produziram. A composição eliminará duplicidades sem apagar informações obrigatórias.

## Outros grupos tipados

| Grupo | Condição |
| --- | --- |
| `purchase` | Informações de pedido, contrato ou empenho exigidas pela operação. |
| `export` | Dados gerais de exportação. |
| `agriculture` | Aquisição ou fornecimento com grupo rural ou de cana aplicável. |
| `intermediary` | Operação realizada por marketplace ou intermediador. |
| `technical_responsible` | Responsável técnico exigido pelo leiaute ou configuração da plataforma. |

Cada grupo deverá possuir contrato tipado, validações e testes próprios antes de ser habilitado. Um campo genérico de extensão não poderá ser usado para contornar essa exigência.

## Precedência por categoria

| Categoria | Ordem e política |
| --- | --- |
| Grupo autenticado | API Key é a única autoridade; qualquer tentativa de informar outro grupo é rejeitada. |
| Ambiente | Credencial e configuração da empresa prevalecem; o payload não pode alternar o ambiente. |
| Emitente | `company_id` seleciona o cadastro do grupo; dados oficiais do cadastro prevalecem. |
| Fatos comerciais | Valor explícito do ERP prevalece; cadastro apenas completa campos cuja política permita. |
| Destinatário | ERP define a identidade da operação; cadastro completa dados sem trocar a pessoa. |
| Produto | Cadastro fiscal vigente prevalece nos campos declarados autoritativos; divergências são expostas. |
| Enquadramento fiscal | Pacote fiscal vigente prevalece; valor do ERP é, no máximo, uma sugestão validável. |
| Cálculos | Motor fiscal prevalece; valor do ERP é usado somente como conferência. |
| Numeração e chave | Plataforma é a única autoridade. |
| Textos legais | Pacote fiscal prevalece; textos comerciais do ERP são adicionados quando permitidos. |
| Configuração da integração | Pode mapear e normalizar, mas não altera fatos, regras fiscais ou autoridade dos campos. |

Nenhum conflito será solucionado silenciosamente. A política específica do campo decidirá entre aceitar, completar, rejeitar ou exigir correção, sempre com rastreabilidade.

## Critérios para enriquecimento automático

Um valor somente poderá ser preenchido automaticamente quando:

1. a matriz permitir aquela origem para o campo;
2. a referência pertencer ao grupo autenticado e estiver ativa;
3. o cadastro ou a regra estiver vigente na data fiscal;
4. houver um único resultado compatível com o contexto;
5. a transformação for determinística e auditável;
6. o preenchimento não inventar um fato comercial;
7. o valor final passar pelas validações do contrato e do cenário.

Se mais de um resultado for possível, a plataforma não escolherá por aproximação.

## Catálogo semântico de erros

| Código | Quando usar |
| --- | --- |
| `FISCAL_FIELD_REQUIRED` | Campo base ou condicional ausente. |
| `FISCAL_FIELD_INVALID` | Valor fora do tipo, formato, tamanho ou domínio aceito. |
| `FISCAL_FIELD_CONFLICT` | Duas fontes autorizadas apresentam valores incompatíveis. |
| `FISCAL_REFERENCE_NOT_FOUND` | Cadastro ou documento referenciado não existe ou não pertence ao grupo. |
| `FISCAL_PRODUCT_DATA_MISSING` | Produto sem dados fiscais suficientes para o cenário. |
| `FISCAL_RULE_NOT_FOUND` | Nenhuma regra vigente resolve o cenário. |
| `FISCAL_RULE_AMBIGUOUS` | Mais de uma regra incompatível resolve o mesmo contexto. |
| `FISCAL_SCENARIO_UNSUPPORTED` | O manifesto necessário não está ativo para o ambiente. |
| `FISCAL_CALCULATION_MISMATCH` | Valor de conferência excede a tolerância do cálculo oficial. |
| `FISCAL_DOCUMENT_NOT_READY` | O documento ainda possui pendências bloqueantes. |

O erro deverá identificar o caminho do campo, a causa e a correção esperada, sem expor segredos ou dados de outro grupo. Status HTTP e envelope público serão definidos na Etapa 6.

Validação interna não é rejeição da SEFAZ. Uma rejeição da autoridade fiscal somente existe depois de uma transmissão efetiva e deverá preservar código, mensagem, protocolo e resposta técnica recebida.

## Rastreabilidade por campo

Para cada decisão relevante, a plataforma deverá conseguir registrar:

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

Também serão preservados o payload original, o resultado do mapeamento, o documento normalizado, o snapshot enriquecido, os conflitos, as versões do contrato e do módulo fiscal e o pacote de schema XML usado.

O snapshot que sustentou uma emissão não será reinterpretado quando cadastros ou regras mudarem.

## Manifesto de cenário fiscal

Cada cobertura fiscal terá um manifesto com:

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

Status permitidos:

```text
planned
implemented
homologated
active
suspended
retired
```

Somente `active` permite emissão em produção. A habilitação será independente entre homologação e produção.

## Critérios para declarar um cenário suportado

Um cenário somente poderá ser ativado quando:

1. todas as dimensões e condições de entrada estiverem definidas;
2. todos os campos aplicáveis tiverem origem, precedência e validação;
3. o pacote fiscal não produzir ausência nem ambiguidade;
4. fórmulas, precisão, tolerâncias e arredondamentos estiverem versionados;
5. o XML gerado validar contra o pacote de schema oficial aplicável;
6. testes positivos, negativos e de fronteira estiverem automatizados;
7. o fluxo estiver validado no ambiente de homologação;
8. fontes oficiais, versões, vigências e limitações estiverem registradas.

Uma empresa compatível utiliza o mesmo cenário sem criar uma versão de API ou regra particular por cliente.

## Catálogo mínimo de testes

Cada cenário deverá cobrir, no mínimo:

- documento mínimo válido;
- documento completo válido;
- ausência de cada campo condicional ativado;
- formatos, tamanhos e domínios inválidos;
- conflito entre payload, cadastro e regra;
- cadastro ou referência inexistente e pertencente a outro grupo;
- produto com cadastro fiscal incompleto;
- destinatário incompatível com a operação;
- regra ausente, ambígua, ainda não vigente ou expirada;
- precisão, arredondamento, rateio e limites numéricos;
- divergência entre valor informado e cálculo oficial;
- cenário ou grupo especial ainda não suportado;
- validação do XML contra o schema oficial selecionado;
- respostas esperadas obtidas em homologação.

Casos reais de segmentos diferentes servirão para testar a abrangência, nunca para criar uma estrutura exclusiva para uma empresa.

## Limites desta matriz

Não fazem parte da Etapa 4:

- fórmulas tributárias concretas de cada pacote fiscal;
- reserva de numeração, idempotência, retentativas e máquina de estados;
- rotas HTTP, envelopes públicos e status HTTP;
- construção física do XML, assinatura, transmissão e contingência;
- armazenamento e proteção de certificados;
- tabelas, índices e migrations.

Esses assuntos pertencem às etapas seguintes do roteiro. A separação evita antecipar decisões que ainda precisam ser discutidas e aprovadas.

## Decisão final da Etapa 4

O `nfe/v1` possui uma matriz semântica independente de cliente e segmento. O ERP fornece fatos comerciais; cadastros completam dados autorizados; pacotes fiscais decidem o enquadramento; o motor fiscal calcula; e a plataforma controla identidade, ambiente, numeração e rastreabilidade.

Campos condicionais são obrigatórios quando a condição ocorre. Cenários incompletos, ambíguos, não homologados ou sem regras vigentes não serão transmitidos nem anunciados como suportados.
