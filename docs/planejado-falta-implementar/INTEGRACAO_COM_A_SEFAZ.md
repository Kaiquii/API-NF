# Integração com a SEFAZ

> Status: planejamento concluído, aguardando implementação.

## Objetivo

Definir como o módulo de NF-e gera, valida, assina, transmite, consulta e armazena os documentos trocados com a SEFAZ, mantendo o processamento assíncrono, idempotente e rastreável.

Esta etapa começa pela NF-e modelo 55. Outros documentos fiscais deverão possuir conectores próprios para suas respectivas autoridades fiscais.

## Referências oficiais

Base consultada em 29 de julho de 2026:

- [Manual de Orientação do Contribuinte da NF-e](https://www.nfe.fazenda.gov.br/portal/exibirArquivo.aspx?conteudo=nebWFce4X9o%3D);
- [Relação oficial de Serviços Web da NF-e](https://www.nfe.fazenda.gov.br/portal/webServices.aspx?AspxAutoDetectCookieSupport=1&tipoConteudo=OUC%2FYVNWZfo%3D);
- [Schemas XML oficiais da NF-e](https://www.nfe.fazenda.gov.br/portal/listaConteudo.aspx?AspxAutoDetectCookieSupport=1&tipoConteudo=BMPFMBoln3w%3D);
- [Notas Técnicas vigentes da NF-e](https://www.nfe.fazenda.gov.br/portal/listaConteudo.aspx?AspxAutoDetectCookieSupport=1&tipoConteudo=04BIflQt1aY%3D).

As versões vigentes não serão fixadas permanentemente neste documento. O módulo deverá registrar os pacotes, versões, fontes e períodos de vigência utilizados em cada processamento.

## Decisão central

A API pública não aguardará a comunicação com a SEFAZ.

```text
API
  -> autentica e identifica o tenant
  -> mapeia, normaliza e valida
  -> registra o documento
  -> reserva a numeração no momento definido
  -> coloca o processamento na fila
  -> retorna 202 Accepted

Worker fiscal
  -> adquire a concessão do documento
  -> carrega o snapshot fiscal imutável
  -> gera e valida o XML
  -> assina com o certificado da empresa
  -> transmite ao autorizador correto
  -> interpreta ou reconcilia o resultado
  -> armazena os artefatos fiscais
  -> atualiza o documento e seu histórico
```

O cliente acompanha o recurso usando os endpoints de consulta definidos na Etapa 6. A conexão HTTP de criação não permanece aberta esperando autorização.

## Limites entre API e worker

### Responsabilidades da API

- validar autenticação e pertencimento ao grupo;
- receber o contrato padrão ou customizado;
- aplicar mapeamento e normalização;
- executar as validações anteriores à emissão;
- aplicar idempotência;
- registrar o documento e seu snapshot;
- reservar a numeração conforme a Etapa 5;
- publicar o trabalho para processamento;
- retornar o recurso e seu status público.

### Responsabilidades do worker

- garantir uma única tentativa ativa por documento;
- usar somente o snapshot fiscal associado à emissão;
- selecionar módulo, schema XML, certificado e autorizador;
- gerar, validar e assinar o XML;
- transmitir e consultar os serviços oficiais;
- classificar respostas e falhas;
- reconciliar resultados desconhecidos;
- persistir tentativas, artefatos e transições;
- agendar retentativas permitidas.

O trabalho publicado na fila deverá carregar identificadores, e não certificados, senhas ou o XML completo. O worker recuperará os dados confiáveis no contexto do tenant e da empresa.

## Organização interna do módulo

O módulo de NF-e será separado em componentes com responsabilidades específicas:

```text
nfe_orchestrator
xml_builder
xml_schema_validator
certificate_provider
xml_signer
sefaz_service_catalog
sefaz_client
sefaz_response_interpreter
nfe_reconciliation_service
fiscal_artifact_store
```

No MVP, esses componentes poderão ser executados no mesmo processo de worker. A separação é lógica e não exige microsserviços independentes.

O orquestrador controla o fluxo, mas não implementa regras tributárias, detalhes criptográficos ou chamadas HTTP diretamente.

## Serviços da NF-e

O primeiro módulo deverá possuir contratos para:

| Serviço | Responsabilidade |
| --- | --- |
| Status do serviço | Verificar disponibilidade técnica do autorizador. |
| Autorização | Transmitir uma NF-e ou lote conforme o leiaute vigente. |
| Retorno de autorização | Consultar o processamento quando houver recibo. |
| Consulta de protocolo | Consultar a situação atual da NF-e pela identidade fiscal. |
| Recepção de evento | Enviar cancelamento e futuros eventos suportados. |
| Inutilização | Solicitar inutilização de faixa de numeração quando aplicável. |

Consulta de cadastro e distribuição de DF-e não fazem parte da primeira entrega de emissão. Poderão ser adicionadas posteriormente como fluxos independentes.

## Catálogo de serviços

URLs da SEFAZ não ficarão espalhadas ou fixadas no código do cliente de comunicação.

O catálogo deverá considerar:

```text
document_type
fiscal_model
state
environment
authorizer
service
service_version
url
effective_from
effective_until
status
source_reference
```

O endpoint será resolvido a partir da empresa, UF, ambiente, serviço e política de autorização. Nenhuma URL ou identificação de autorizador será aceita do payload do cliente.

Alterações no catálogo serão versionadas, testadas e auditadas. Uma tentativa deverá registrar a versão e o endpoint efetivamente utilizados.

Homologação e produção terão entradas separadas. Uma configuração de homologação não poderá direcionar chamadas para produção.

## Pacotes de schema XML

Schemas oficiais serão tratados como pacotes versionados e imutáveis.

Cada pacote deverá registrar:

```text
package_id
document_type
layout_version
package_version
effective_from
effective_until
status
source_reference
checksum
```

Um pacote novo passa por testes antes de se tornar ativo. Sua ativação não substitui nem remove o pacote usado em documentos anteriores.

O manifesto do cenário fiscal, definido na Etapa 4, determina qual pacote é aplicável. O worker não escolhe a versão apenas por ela ser a mais recente.

## Snapshot fiscal

O XML será construído a partir do snapshot fiscal congelado depois das validações.

O worker não poderá consultar versões atuais de produto, destinatário ou regra para reinterpretar uma emissão já enfileirada. Se o documento precisar ser corrigido antes da emissão, deverá voltar pelo fluxo permitido e produzir um novo snapshot antes da reserva e transmissão.

O snapshot deverá identificar:

- contrato público e interno;
- mapeamento aplicado;
- empresa e ambiente;
- dados normalizados e enriquecidos;
- pacote fiscal;
- pacote de schema XML;
- versão do módulo;
- decisão e cálculos fiscais;
- data fiscal usada na resolução.

## Geração do XML

O `xml_builder` transforma o snapshot em XML do leiaute selecionado.

Requisitos:

- geração determinística para o mesmo snapshot e identidade fiscal;
- namespaces, ordem, cardinalidade e formatos conforme o schema;
- codificação definida pelo leiaute;
- ausência de campos que não se apliquem ao cenário;
- preservação da precisão calculada pelo motor fiscal;
- nenhuma consulta a dados comerciais externos durante a geração;
- identificação da versão do gerador.

O XML não será montado com concatenação livre de strings. A implementação utilizará estruturas tipadas compatíveis com o leiaute do módulo.

## Validação do XML

A validação contra o schema oficial ocorrerá antes da transmissão. Quando a assinatura alterar a estrutura validada, o documento assinado também deverá passar pela validação estrutural aplicável.

Uma falha local de schema:

- não transmite o documento;
- não é registrada como rejeição da SEFAZ;
- encerra a tentativa com erro não retentável automaticamente;
- identifica o elemento e o pacote de schema;
- preserva informações suficientes para diagnóstico sem registrar o XML em logs comuns.

## Certificado e assinatura

O certificado A1 pertence à empresa emitente.

O módulo desta etapa consumirá uma abstração `certificate_provider`, responsável por entregar ao assinador uma credencial válida para uso temporário. O módulo da SEFAZ não receberá senha em arquivo de configuração, payload, fila ou log.

Antes da assinatura, deverão ser confirmados:

- certificado ativo para a empresa;
- período de validade;
- finalidade e cadeia compatíveis;
- vínculo permitido com o emitente;
- ambiente e operação autorizados;
- política de seleção quando houver rotação.

O assinador deverá:

- aplicar o padrão exigido pelo leiaute;
- devolver o XML assinado e metadados não secretos;
- limitar a permanência do material sensível em memória;
- nunca registrar certificado, chave privada ou senha em logs.

Criptografia em repouso, cofre de segredos, acesso administrativo, rotação e recuperação serão definidos na Etapa 8.

## Tentativa de emissão

Antes de transmitir, o worker deverá:

1. adquirir a concessão exclusiva do documento;
2. confirmar o estado `queued` ou outro estado permitido;
3. criar uma `issuance_attempt` imutável;
4. confirmar numeração e identidade fiscal;
5. resolver schema, autorizador, serviço e certificado;
6. gerar e validar o XML;
7. assinar e persistir o artefato que será transmitido;
8. registrar o hash do XML;
9. iniciar a chamada externa com timeout controlado.

Uma nova tentativa não sobrescreve a anterior. O mesmo documento não poderá possuir duas transmissões concorrentes.

## Comunicação com a SEFAZ

O `sefaz_client` será responsável somente pela comunicação técnica:

- protocolo e envelope exigidos pelo serviço;
- autenticação mútua quando aplicável;
- headers e versões oficiais;
- timeouts de conexão, envio e resposta;
- limite de tamanho;
- captura controlada da resposta;
- métricas técnicas sem conteúdo fiscal sensível.

Ele não decide se uma rejeição deve ser corrigida, se uma nota pode ser retransmitida ou qual regra tributária se aplica.

Chamadas deverão possuir limites e proteção contra repetição excessiva. Consultas de recibo e reconciliação usarão intervalos controlados, sem pressionar o autorizador em ciclos curtos.

## Resposta de autorização

O fluxo deverá aceitar:

```text
resultado definitivo na resposta
ou
recibo para consulta posterior
```

Quando houver resultado definitivo, o interpretador classifica e persiste a resposta.

Quando houver recibo:

1. o recibo é armazenado na tentativa;
2. o documento permanece `processing`;
3. uma consulta de retorno é agendada;
4. cada consulta é registrada;
5. o resultado final atualiza a tentativa e o documento.

O worker não ficará bloqueado aguardando indefinidamente.

## Classificação das respostas

Os códigos oficiais serão preservados e mapeados para categorias internas estáveis:

| Categoria interna | Tratamento |
| --- | --- |
| `authorized` | Armazenar protocolo e XML processado; concluir o documento. |
| `rejected` | Preservar código e mensagem; não retentar automaticamente. |
| `temporarily_unavailable` | Agendar retentativa conforme política. |
| `authorization_unknown` | Iniciar reconciliação antes de qualquer retransmissão. |
| `schema_error` | Corrigir implementação, regra ou pacote; não retentar automaticamente. |
| `certificate_error` | Exigir correção da credencial; não retentar automaticamente. |
| `communication_error` | Classificar pelo momento da falha para decidir entre retentativa e reconciliação. |
| `unexpected_response` | Preservar resposta, interromper repetição e encaminhar para análise. |

O catálogo de códigos e seus significados será versionado por serviço e leiaute. Um código desconhecido não será tratado automaticamente como indisponibilidade.

## Resultado de autorização desconhecido

Uma falha depois do início da transmissão não prova que a SEFAZ deixou de processar a NF-e.

Situações como timeout, conexão interrompida ou queda do worker depois do envio levam a:

```text
processing
  -> authorization_unknown
  -> reconciling
```

O reconciliador usará os mecanismos oficiais disponíveis, como consulta de protocolo ou consulta do recibo existente.

A reconciliação poderá terminar em:

- `authorized`;
- `rejected`;
- `retry_scheduled`, quando houver confirmação de que uma nova tentativa é segura;
- `processing_failed`, quando não houver resolução automática segura.

É proibido, durante resultado desconhecido:

- reservar outro número;
- criar outro documento para a mesma solicitação;
- retransmitir por timeout sem consulta;
- substituir o XML transmitido;
- declarar rejeição sem resposta da autoridade.

## Política de retentativas

Somente falhas temporárias e classificadas poderão gerar retentativa automática.

Cada política deverá definir:

```text
service
error_category
maximum_attempts
initial_interval
maximum_interval
backoff
jitter
reconciliation_required
```

Rejeição fiscal, XML inválido, certificado inválido, configuração ausente e resposta desconhecida não resolvida não geram retransmissão automática.

Ao esgotar o limite, o documento vai para `processing_failed`. Uma ação operacional posterior cria nova tentativa e preserva todo o histórico.

## Persistência dos artefatos

Cada tentativa deverá manter, conforme sua fase:

- identificador do documento e da tentativa;
- snapshot fiscal ou referência imutável;
- XML assinado transmitido;
- hash do XML;
- versão do módulo e do schema;
- catálogo, autorizador, serviço e endpoint usados;
- horários de início, envio, resposta e conclusão;
- recibo e protocolo;
- código e mensagem oficiais;
- categoria interna;
- resposta técnica necessária para auditoria;
- XML processado com protocolo quando autorizado;
- vínculo com consultas e reconciliações.

O XML autorizado e seu protocolo serão imutáveis.

XML, certificado, senha e conteúdo fiscal completo não serão gravados em logs. Criptografia, retenção e permissões sobre os artefatos serão definidas na Etapa 8.

## Cancelamento e outros eventos

Cancelamento será uma operação própria vinculada a um documento autorizado.

```text
fiscal_document autorizado
  -> fiscal_event
      -> event_attempt
      -> autorizado ou rejeitado
```

Uma solicitação de cancelamento não altera o documento para `cancelled` antes da confirmação da SEFAZ. Rejeição ou falha do evento preserva o documento como `authorized`.

Cada evento terá:

- idempotência própria;
- regras de elegibilidade e prazo;
- XML, assinatura e schema próprios;
- tentativa e histórico independentes;
- protocolo e retorno oficial;
- reconciliação quando o resultado for desconhecido.

Eventos além do cancelamento serão adicionados somente quando seus contratos forem planejados, implementados e homologados.

## Inutilização de numeração

A inutilização será um fluxo operacional separado da autorização.

Ela deverá:

- identificar empresa, ambiente, modelo, série e faixa;
- validar se os números podem ser inutilizados;
- exigir justificativa;
- possuir idempotência;
- gerar, validar e assinar sua solicitação;
- armazenar protocolo e resposta;
- registrar o administrador ou processo responsável.

Um número reservado não será reutilizado automaticamente. A necessidade de inutilização será decidida pelo módulo e pelas regras fiscais aplicáveis.

## Contingência

A arquitetura reconhecerá os autorizadores de contingência aplicáveis, incluindo SVC-AN e SVC-RS, conforme a distribuição oficial por UF.

Contingência não será ativada após um único timeout. Antes da troca, a plataforma deverá diferenciar:

- falha interna;
- certificado inválido;
- problema de conectividade local;
- indisponibilidade do autorizador;
- resultado de autorização desconhecido;
- indisponibilidade confirmada e elegível para contingência.

A política deverá registrar:

```text
state
normal_authorizer
contingency_authorizer
activation_mode
activated_at
activated_by
reason
ended_at
effective_rules
```

Regras obrigatórias:

- resultado desconhecido deve ser reconciliado antes de uma decisão que possa duplicar a emissão;
- mudança de modalidade deverá aplicar as regras oficiais de identidade e emissão;
- número fiscal não será devolvido ou substituído silenciosamente;
- XML, chave e tentativa anteriores serão preservados;
- entrada e saída da contingência serão auditadas;
- homologação e produção terão habilitações independentes.

Para o primeiro MVP, a emissão normal será implementada e homologada antes da ativação produtiva da contingência. A contingência permanecerá planejada na arquitetura e será liberada somente após testes específicos.

## Isolamento multi-tenant

O worker deverá obter o grupo e o schema a partir do documento registrado e das referências internas confiáveis.

Ele nunca aceitará `schema_name`, certificado, endpoint ou `group_id` arbitrário vindo da mensagem da fila.

Antes de usar empresa, certificado ou documento, confirmará o pertencimento ao mesmo grupo. Uma tentativa não poderá consultar artefatos de outro schema.

## Observabilidade mínima desta etapa

O processamento deverá produzir métricas técnicas sem expor conteúdo fiscal:

- tentativas por serviço, autorizador e ambiente;
- duração das chamadas;
- autorizações e rejeições por categoria;
- falhas temporárias;
- resultados desconhecidos;
- reconciliações pendentes;
- retentativas agendadas;
- erros de schema e certificado.

O desenho completo de dashboards, alertas, webhooks, suporte e operação pertence à Etapa 10.

## Estratégia de testes

### Testes do XML

- geração determinística;
- casos mínimos e completos;
- validação nos schemas selecionados;
- namespaces, formatos e cardinalidades;
- assinatura válida;
- incompatibilidade entre cenário e schema;
- atualização controlada de pacote.

### Testes do cliente SEFAZ

- autorização definitiva;
- autorização com recibo;
- consulta de recibo ainda em processamento;
- consulta com resultado final;
- rejeição fiscal;
- indisponibilidade temporária;
- timeout antes do envio;
- timeout durante ou depois do envio;
- resposta incompleta ou desconhecida;
- endpoint ou versão incompatível.

### Testes do orquestrador

- uma única tentativa ativa;
- worker interrompido antes da transmissão;
- worker interrompido depois da transmissão;
- recuperação da concessão expirada;
- reconciliação antes de retransmitir;
- retentativa com backoff;
- limite de tentativas;
- preservação do XML e da identidade fiscal;
- transições inválidas recusadas.

### Testes de eventos e contingência

- cancelamento autorizado e rejeitado;
- resultado desconhecido do evento;
- inutilização válida e inválida;
- ativação e encerramento da contingência;
- UF direcionada ao autorizador correto;
- impedimento de contingência durante resultado desconhecido;
- separação entre homologação e produção.

### Homologação

Cada cenário habilitado deverá ser exercitado no ambiente de homologação com:

- certificado e empresa compatíveis;
- autorizador correspondente à UF;
- schema e Nota Técnica vigentes;
- casos autorizados e rejeitados;
- consulta de recibo e protocolo;
- eventos incluídos no escopo;
- evidências, versões e limitações registradas.

Mocks e testes locais não substituem homologação.

## Escopo recomendado para o primeiro MVP

O primeiro recorte da NF-e modelo 55 incluirá:

```text
emissão normal
certificado A1
geração e validação do XML
assinatura
autorização
retorno de autorização
consulta pela chave
reconciliação
cancelamento
inutilização
armazenamento do XML autorizado
```

O suporte produtivo a cada UF e cenário dependerá de implementação, testes e homologação.

Contingência será implementada e ativada depois que o fluxo normal estiver estável, sem exigir alteração na arquitetura definida nesta etapa.

## Limites desta etapa

Esta etapa não define:

- algoritmo e infraestrutura de criptografia dos certificados;
- cofre de segredos e permissões administrativas;
- política completa de retenção e descarte;
- tabelas, índices, fila e formato físico das concessões;
- provedor de armazenamento;
- dashboards, webhooks e procedimentos de suporte;
- cronograma técnico de implementação.

Essas decisões pertencem às Etapas 8, 9, 10 e 11.

## Resultado da etapa

A integração da NF-e com a SEFAZ será assíncrona e executada por worker. O módulo terá geração, validação, assinatura, catálogo de serviços, comunicação, interpretação, reconciliação e armazenamento separados por responsabilidade.

Nenhuma falha de comunicação autorizará retransmissão cega. Resultados desconhecidos serão reconciliados, respostas oficiais serão preservadas e cada tentativa ficará vinculada ao XML, schema, certificado, autorizador e versões utilizados.

O fluxo normal será a primeira entrega. Cancelamento e inutilização fazem parte do MVP planejado, enquanto contingência será preparada desde a arquitetura e ativada em produção somente depois de implementação e homologação específicas.
