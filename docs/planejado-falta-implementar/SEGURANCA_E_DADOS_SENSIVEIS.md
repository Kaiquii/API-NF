# Segurança e dados sensíveis

> Status: planejamento concluído, aguardando implementação.

## Objetivo

Definir a base de segurança necessária para que a API fiscal possa ser implementada e levada à produção com proteção proporcional aos riscos do produto, sem antecipar controles corporativos que ainda não sejam necessários.

Esta etapa protege todo o ciclo do documento fiscal:

```text
ERP ou integração
  -> API
  -> payload original
  -> modelo fiscal interno
  -> banco e fila
  -> worker
  -> certificado A1
  -> SEFAZ
  -> XML autorizado
  -> consulta ou download pelo grupo
```

As decisões devem garantir confidencialidade, integridade, disponibilidade, isolamento entre grupos, rastreabilidade, recuperação e conformidade com as obrigações fiscais e de proteção de dados.

## Estado da etapa

As decisões técnicas, fiscais operacionais e de proteção de dados necessárias ao planejamento foram consolidadas. O projeto adotará políticas conservadoras para o primeiro MVP e exigirá revisão jurídica e fiscal profissional antes da ativação do primeiro cliente em produção.

Essa revisão futura é um portão de produção, não uma indefinição arquitetural. Caso identifique obrigação diferente, a política será versionada antes da entrada em produção, sem alterar os princípios centrais desta etapa.

## Escopo desta etapa

Esta etapa define:

- classificação dos dados e ativos;
- responsabilidades relacionadas à LGPD;
- proteção e ciclo de vida do certificado A1;
- gestão de segredos e chaves criptográficas;
- segurança das API keys;
- autenticação e autorização administrativas;
- requisitos de isolamento multi-tenant;
- proteção de payloads e artefatos fiscais;
- política de logs, mascaramento e auditoria;
- requisitos mínimos de infraestrutura;
- retenção e descarte;
- backup e recuperação;
- resposta a incidentes;
- requisitos de desenvolvimento seguro;
- critérios e testes mínimos de segurança.

## Limites desta etapa

Esta etapa não define:

- tabelas, colunas, índices e migrations definitivos, reservados à Etapa 9;
- ferramenta de fila, formato físico das concessões e detalhes de concorrência;
- dashboards, plataforma de observabilidade e operação de suporte, reservados à Etapa 10;
- provedor de nuvem ou produto comercial definitivo;
- cronograma e ordem técnica de implementação, reservados à Etapa 11;
- controles corporativos avançados sem justificativa de risco ou contrato.

## Princípio de proporcionalidade

Cada controle será classificado em uma das categorias abaixo.

### Obrigatório para a primeira produção

Controle sem o qual existe risco inaceitável de fraude, exposição de dados, comprometimento de certificado, acesso entre tenants, perda de documento fiscal ou impossibilidade de investigação e recuperação.

### Arquitetura preparada

Capacidade que não precisa ser implementada completamente no primeiro MVP, mas que não pode ser bloqueada por uma decisão estrutural difícil de reverter.

### Evolução futura

Controle que somente será implementado quando volume, clientes, contratos, regulação ou avaliação de risco justificarem a complexidade.

## Princípios obrigatórios

1. Nenhum segredo, certificado ou chave privada será armazenado em texto puro.
2. O tenant sempre será derivado de uma identidade autenticada, nunca de um valor arbitrário enviado pelo cliente.
3. Toda operação verificará o pertencimento do recurso ao grupo e, quando aplicável, à empresa.
4. Logs não conterão segredos nem conteúdo fiscal completo.
5. Acesso a ativos críticos seguirá privilégio mínimo e será auditável.
6. Dados e artefatos terão proteção de integridade, não apenas de confidencialidade.
7. Produção, homologação e desenvolvimento terão identidades e segredos separados.
8. Controles críticos possuirão testes ou evidências verificáveis.
9. Falhas de segurança deverão resultar em bloqueio seguro, sem continuar com valores padrão ou proteções desativadas.
10. A aplicação não implementará algoritmos criptográficos próprios.

## Referências de segurança e conformidade

As decisões e a implementação deverão considerar:

- Lei Geral de Proteção de Dados Pessoais e orientações da ANPD;
- regras vigentes da ICP-Brasil para certificados digitais;
- Manual de Orientação do Contribuinte e documentação oficial da NF-e;
- OWASP Application Security Verification Standard;
- OWASP API Security Top 10;
- práticas de gestão de chaves e riscos do NIST, como referência técnica não vinculante.

Versões específicas dessas referências deverão ser registradas no plano de implementação e atualizadas antes de cada entrada em produção.

## Classificação dos dados e ativos

### Níveis

| Classe | Exemplos | Proteção mínima |
| --- | --- | --- |
| Pública | Documentação pública, schemas oficiais e catálogos públicos da SEFAZ | Integridade e versionamento. |
| Interna | Configurações não sensíveis, métricas agregadas e documentação operacional | Acesso autenticado e necessidade de conhecimento. |
| Confidencial | XML, payload, CPF, endereço, dados do destinatário, valores e regras comerciais | Criptografia em trânsito e repouso, acesso restrito, mascaramento e retenção definida. |
| Crítica | PFX, senha do certificado, chave privada, API key completa, chaves mestras e credenciais de infraestrutura | Cofre ou provedor criptográfico, privilégio mínimo, acesso auditado e proibição em logs. |

### Catálogo de dados

Todo campo ou artefato persistido deverá possuir, no mínimo:

- classificação;
- finalidade;
- origem;
- proprietário ou responsável;
- serviços e perfis autorizados;
- prazo ou critério de retenção;
- permissão ou proibição de aparecer em logs;
- necessidade de criptografia de aplicação;
- comportamento em backup e descarte.

O catálogo físico será associado às tabelas e armazenamentos definidos na Etapa 9.

## Responsabilidades de proteção de dados

Como referência arquitetural:

- o grupo atuará, em regra, como controlador dos dados fiscais que enviar;
- a plataforma atuará, em regra, como operadora desse tratamento;
- a plataforma poderá atuar como controladora dos dados tratados para suas próprias finalidades de segurança, administração e faturamento;
- provedores de infraestrutura, armazenamento, comunicação e observabilidade poderão atuar como suboperadores.

O contrato ou termo de uso deverá identificar:

- quando o grupo atua como controlador dos dados;
- quando a plataforma atua como operadora;
- situações em que a plataforma possa assumir decisões próprias de tratamento;
- fornecedores que atuem como suboperadores;
- localização e transferência dos dados;
- responsabilidades por atendimento a titulares e incidentes;
- canal de contato e responsáveis internos.

Para o MVP, ficam adotadas as seguintes responsabilidades:

- o grupo decide a finalidade comercial e fiscal e responde pela legitimidade e exatidão dos dados enviados;
- a plataforma trata os dados fiscais para executar as instruções de emissão, consulta, eventos e armazenamento;
- a plataforma decide sobre os tratamentos próprios indispensáveis à segurança, prevenção de abuso, administração e faturamento;
- o grupo é responsável por fornecer certificado válido e possuir autorização para utilizá-lo;
- a plataforma é responsável por proteger o certificado enquanto estiver sob sua custódia;
- fornecedores somente poderão tratar os dados para prestar o serviço contratado e sob requisitos equivalentes de segurança.

Essas definições deverão ser refletidas no contrato antes da primeira produção. A arquitetura não assumirá que todo dado fiscal é público nem que obrigações fiscais eliminam os deveres de proteção de dados pessoais.

### Suboperadores

A plataforma manterá uma relação versionada dos fornecedores que tratem dados em seu nome, contendo finalidade, categoria do serviço e localização relevante.

Requisitos:

- autorização geral do grupo prevista contratualmente;
- comunicação de mudança relevante;
- obrigação de segurança e confidencialidade equivalente;
- proibição de uso próprio incompatível;
- verificação de transferência internacional quando aplicável;
- responsabilidade contratual da plataforma pela gestão do fornecedor.

O primeiro MVP poderá usar uma página ou anexo versionado; não será necessário construir um portal de fornecedores.

### Encerramento do relacionamento

O encerramento seguirá:

```text
bloquear novas emissões
  -> revogar API keys
  -> bloquear certificado
  -> permitir exportação dos XMLs por 30 dias
  -> encerrar a janela de exportação
  -> eliminar dados sem obrigação de retenção
  -> manter dados legalmente obrigatórios com uso bloqueado
  -> expirar cópias conforme o ciclo de backup
  -> registrar a conclusão em auditoria
```

Dados sujeitos a fiscalização, processo, disputa ou outra retenção legal entrarão em `legal hold` até a liberação formal. A plataforma não prometerá eliminação imediata das cópias já existentes em backups; elas permanecerão inacessíveis ao uso normal e expirarão no ciclo documentado.

## Modelo de ameaças mínimo

O modelo de ameaças deverá ser mantido por fluxo e atualizado quando a arquitetura mudar.

### Ativos principais

- certificado A1 e sua chave privada;
- senha do certificado;
- API keys;
- chaves de criptografia;
- payloads recebidos;
- modelo fiscal interno;
- XML assinado e autorizado;
- protocolo e chave de acesso;
- dados pessoais de emitentes e destinatários;
- configurações fiscais e de integração;
- trilhas de auditoria;
- backups.

### Fronteiras de confiança

- ERP para API;
- API para Postgres;
- API para fila;
- fila para worker;
- worker para provedor de certificado;
- worker para SEFAZ;
- aplicação para armazenamento de objetos;
- painel administrativo para endpoints internos;
- aplicação para observabilidade e suporte.

### Riscos que devem ser tratados

- comprometimento de API key;
- uso indevido do certificado para assinar documentos;
- acesso a dados de outro grupo;
- alteração ou substituição de XML autorizado;
- exposição de payload ou XML em logs;
- injeção e manipulação de mapeamentos;
- acesso administrativo excessivo;
- vazamento por backup ou ambiente de desenvolvimento;
- SSRF por webhooks, URLs ou integrações;
- dependência ou imagem comprometida;
- indisponibilidade ou perda dos artefatos fiscais;
- retransmissão indevida após falha de comunicação;
- uso de credenciais de homologação em produção ou o inverso.

## Certificado digital A1

O certificado A1 é o ativo de maior impacto da plataforma, pois permite assinar documentos em nome da empresa emitente.

### Cadastro

O cadastro deverá ocorrer somente por fluxo administrativo autenticado e autorizado.

No recebimento, a aplicação deverá validar:

- formato e integridade do PFX;
- senha informada;
- cadeia de certificação;
- CNPJ associado;
- período de validade;
- finalidade compatível;
- pertencimento à empresa e ao grupo;
- ambiente no qual o certificado poderá ser usado.

Certificado inválido, vencido, incompatível ou pertencente a outro CNPJ será recusado.

### Armazenamento

- PFX e senha nunca serão persistidos em texto puro.
- PFX e senha formarão um único pacote de credencial, criptografado antes do armazenamento.
- A chave mestra ficará fora do Postgres e fora do repositório da aplicação.
- Produção usará KMS, cofre de segredos ou serviço equivalente com controle de acesso e auditoria.
- A criptografia de aplicação será autenticada, impedindo alteração silenciosa do conteúdo.
- O contexto criptográfico deverá vincular ambiente, grupo, empresa e identificador do certificado.
- O pacote cifrado será armazenado no Postgres, no schema do grupo, junto aos metadados e à chave de dados protegida.
- Backups nunca conterão uma cópia adicional em texto puro.

A forma física das colunas e restrições será definida na Etapa 9. A implementação será acessada por uma abstração `certificate_provider`, sem acoplar o domínio fiscal a um provedor comercial.

### Formato criptográfico

O certificado usará criptografia envelope:

1. gerar uma chave de dados aleatória de 256 bits para cada versão do certificado;
2. montar o pacote contendo PFX e senha;
3. criptografar o pacote com AES-256-GCM;
4. proteger a chave de dados usando o provedor de KMS ou cofre;
5. persistir somente pacote cifrado, chave de dados protegida e metadados;
6. autenticar o contexto formado por ambiente, grupo, empresa e certificado.

Metadados mínimos:

```text
algorithm
encryption_version
key_provider
master_key_id
wrapped_data_key
nonce
ciphertext
group_id
company_id
certificate_id
environment
```

Bibliotecas criptográficas consolidadas serão usadas para executar essas operações. Não haverá algoritmo, modo de operação ou formato criptográfico inventado pela aplicação.

### Uso pelo worker

O trabalho publicado na fila conterá apenas identificadores.

Antes de assinar, o worker deverá:

1. resolver internamente o tenant;
2. carregar o documento e a empresa no schema correto;
3. confirmar o vínculo entre documento, empresa e certificado;
4. confirmar ambiente, status e validade;
5. obter temporariamente a credencial pelo `certificate_provider`;
6. assinar e descartar as referências ao material sensível assim que possível.

O certificado, a senha e a chave privada não poderão ser:

- enviados na fila;
- gravados em arquivo temporário;
- retornados por endpoint de consulta;
- incluídos em log, erro, métrica ou tracing;
- enviados a serviços de observabilidade;
- disponibilizados para download depois do cadastro.

### Ciclo de vida

Cada certificado possuirá:

- empresa proprietária;
- CNPJ;
- número de série;
- emissor;
- início e fim da validade;
- ambiente permitido;
- status;
- data de cadastro;
- responsável pelo cadastro;
- data e motivo de substituição, bloqueio ou revogação.

Documentos históricos continuarão associados à versão e ao certificado utilizados, sem depender de o certificado ainda estar ativo.

## Gestão de chaves criptográficas e segredos

### Requisitos

- chaves mestras serão separadas por ambiente;
- somente identidades de serviço autorizadas poderão solicitar criptografia ou descriptografia;
- API, worker, administração e migrations terão permissões distintas;
- acesso a operações criptográficas críticas será auditado;
- rotação de chave será prevista sem exigir descriptografia massiva imediata;
- indisponibilidade do provedor bloqueará a operação de forma segura;
- segredos não serão fornecidos por payload, fila ou parâmetros de linha de comando;
- segredos não serão gravados no repositório, imagem ou artefato de build.

O desenvolvimento local poderá usar um provedor simplificado, explicitamente identificado e impossível de habilitar em produção.

O produto comercial definitivo não precisa ser escolhido nesta etapa. Qualquer opção de produção deverá oferecer chaves separadas por ambiente, controle de acesso por identidade de serviço, auditoria, rotação e proteção da chave mestra fora do banco.

## API keys

### Geração e armazenamento

- segredo gerado com `crypto/rand` e pelo menos 32 bytes aleatórios;
- formato com identificador público e segredo confidencial;
- segredo completo exibido somente na criação ou rotação;
- armazenamento somente de hash ou HMAC apropriado;
- comparação em tempo constante;
- vínculo obrigatório com grupo e ambiente;
- separação entre homologação e produção;
- rate limit e quota por chave, grupo e IP;
- revogação com efeito imediato.

### Rotação planejada

A regra normal será uma chave principal ativa por grupo e ambiente. Para evitar indisponibilidade durante uma troca planejada, poderá existir uma segunda chave ativa por uma janela curta de rotação:

```text
criar nova chave
  -> atualizar o ERP
  -> confirmar uso da nova chave
  -> revogar a chave anterior
```

A janela padrão será de 72 horas, configurável até o limite máximo de sete dias. A chave anterior expirará automaticamente ao final da janela, mesmo que o administrador não conclua manualmente a rotação.

Em caso de perda ou suspeita de comprometimento, o reset emergencial revogará imediatamente todas as credenciais anteriores.

O planejamento de autenticação foi harmonizado com essa regra durante a Etapa 8.

## Autenticação e autorização administrativas

Mesmo que o painel administrativo seja implementado depois, os endpoints internos deverão seguir:

- contas individuais, sem compartilhamento;
- MFA obrigatório;
- sessão com expiração;
- reautenticação para ações críticas;
- controle de acesso baseado em função;
- privilégio mínimo;
- auditoria de toda operação administrativa relevante;
- proibição de usar API key de grupo em endpoint administrativo.

Perfis iniciais:

| Perfil | Permissões principais |
| --- | --- |
| Suporte | Consultar status e metadados mascarados, sem certificado, segredo ou payload integral. |
| Operador fiscal | Consultar documentos, erros e XML autorizado e executar operações fiscais permitidas. |
| Administrador de grupo | Gerenciar empresas e integrações do grupo. |
| Administrador de credenciais | Cadastrar ou substituir certificados e criar, rotacionar ou revogar API keys. |
| Administrador de segurança | Gerenciar permissões, bloquear acessos e investigar incidentes. |

Uma pessoa poderá acumular perfis em uma equipe pequena, mas as permissões permanecerão separadas tecnicamente e serão concedidas de forma explícita.

Ações críticas, como cadastrar ou substituir certificado, resetar credenciais e alterar permissões, exigirão permissão específica.

## Isolamento multi-tenant

O tenant será sempre derivado da credencial autenticada.

Não serão fontes confiáveis:

- `group_id` enviado pelo cliente;
- `schema_name` recebido em payload, header ou query;
- nome de schema informado na fila;
- endpoint SEFAZ fornecido pelo payload;
- identificador de empresa sem validação de pertencimento.

O fluxo obrigatório será:

```text
credencial
  -> grupo
  -> schema controlado pela plataforma
  -> empresa
  -> integração, documento ou certificado
  -> autorização da operação
```

Requisitos:

- `schema_name` obtido somente de `platform.groups`;
- seleção de schema no escopo da transação;
- uso de configuração local, como `SET LOCAL search_path`;
- validação de pertencimento em toda operação por identificador;
- nenhuma busca automática do recurso em outros schemas;
- identidade e permissões próprias para o worker;
- usuário da aplicação sem permissão para criar schemas ou executar migrations;
- limpeza segura do contexto ao reutilizar conexões do pool.

## Proteção de XMLs e payloads

### XML autorizado

O XML assinado e autorizado será preservado no formato original, junto ao protocolo correspondente.

Requisitos:

- armazenamento privado;
- criptografia em repouso;
- hash SHA-256 para conferência de integridade;
- metadados que identifiquem documento, versão e ambiente;
- acesso somente após autorização por tenant e recurso;
- ausência de URL pública permanente;
- proteção contra exclusão acidental;
- backup criptografado;
- registro de acesso e download.

O armazenamento de objetos privado é a opção preferencial. O Postgres deverá guardar metadados, hash e referência, sujeito à decisão física da Etapa 9.

No primeiro MVP, a criptografia gerenciada pelo armazenamento, usando chave vinculada ao ambiente, será suficiente para os XMLs. Criptografia adicional na camada da aplicação ficará preparada, mas somente será exigida se o risco, o contrato ou o provedor de infraestrutura justificar.

### Payloads

- payload original será mantido somente quando necessário para rastreabilidade, idempotência, suporte ou obrigação definida;
- dados pessoais e fiscais terão criptografia e acesso restrito;
- tamanho, profundidade e tempo de processamento serão limitados;
- conteúdo recebido continuará não confiável mesmo depois do mapeamento;
- payload não poderá controlar schema, caminho de arquivo, endpoint ou código executável;
- dados de produção não serão copiados para desenvolvimento.

## Logs, erros, métricas e tracing

### Conteúdo proibido

Nunca registrar:

- API key completa;
- header `Authorization`;
- PFX, senha, chave privada ou chave criptográfica;
- XML completo;
- payload completo;
- string de conexão;
- segredo de webhook;
- stack trace em resposta externa.

### Conteúdo sujeito a mascaramento

- CPF e CNPJ;
- e-mail e telefone;
- endereço;
- dados bancários ou de pagamento;
- valores comerciais quando não forem necessários ao diagnóstico;
- campos livres que possam conter dados pessoais.

O mascaramento será centralizado e ativado por padrão. Logs de depuração não poderão desabilitar essas proteções em produção.

### Erros externos

Respostas ao cliente poderão informar código, categoria, campo e orientação de correção, mas não revelarão:

- detalhes de infraestrutura;
- nomes físicos de schemas;
- consultas SQL;
- conteúdo de outro tenant;
- segredos;
- stack traces.

## Auditoria

Auditoria de segurança será separada dos logs operacionais.

Eventos obrigatórios:

- cadastro, uso, substituição, bloqueio e revogação de certificado;
- criação, rotação, reset e revogação de API key;
- acesso e download de XML;
- alteração de permissões;
- ações administrativas;
- tentativas de acesso entre tenants;
- falhas relevantes de autenticação e autorização;
- operações de descriptografia;
- mudanças em configurações fiscais críticas.

Cada evento deverá registrar:

- ator ou identidade de serviço;
- tenant e empresa, quando aplicável;
- ação;
- tipo e identificador do recurso;
- data e origem;
- resultado;
- motivo administrativo, quando exigido;
- identificador de correlação.

A auditoria não armazenará o segredo nem o conteúdo sensível do recurso. Sua persistência deverá dificultar alteração ou exclusão não autorizada.

## Infraestrutura mínima para produção

- HTTPS em toda interface externa;
- Postgres e armazenamento sem acesso público direto;
- credenciais de serviço com privilégio mínimo;
- produção, homologação e desenvolvimento separados;
- segredos e chaves diferentes por ambiente;
- dados produtivos proibidos em desenvolvimento;
- limites de tamanho, tempo e recursos nas entradas;
- destinos externos controlados;
- correções de segurança com prioridade definida;
- backups criptografados;
- acesso administrativo restrito e auditado.

O worker poderá ser executado no mesmo projeto e infraestrutura da API no MVP. Segurança não exige microsserviços, orquestrador de containers ou múltiplas regiões na primeira entrega.

## Comunicação externa e SSRF

- chamadas à SEFAZ usarão apenas endpoints do catálogo interno versionado;
- o cliente não poderá informar livremente URLs usadas pelo worker;
- redirecionamentos externos não serão seguidos sem validação;
- toda chamada possuirá TLS, timeout, limite de resposta e validação de conteúdo;
- futuros webhooks terão validação de destino e proteção contra redes internas e metadados de infraestrutura;
- respostas externas continuarão sendo tratadas como não confiáveis.

## Retenção e descarte

Não haverá um único prazo para todos os dados.

Política inicial:

| Dado ou artefato | Retenção inicial |
| --- | --- |
| XML autorizado e protocolo | Seis anos a partir da emissão, ou prazo maior quando houver obrigação específica ou `legal hold`. |
| Eventos fiscais, como cancelamento, carta de correção e inutilização | Seis anos a partir do evento, ou prazo maior aplicável. |
| Snapshot fiscal e decisões aplicadas | Seis anos, acompanhando o documento correspondente. |
| Payload original | 180 dias após o estado final, salvo disputa, incidente ou obrigação específica. |
| Payload normalizado | Seis anos quando necessário para reproduzir e auditar a emissão. |
| Respostas relevantes da SEFAZ | Seis anos, acompanhando o documento ou evento correspondente. |
| Logs operacionais | 90 dias disponíveis e até um ano em arquivo protegido. |
| Auditoria de segurança | Cinco anos. |
| Registros de incidentes com dados pessoais | No mínimo cinco anos. |
| Backups diários | 35 dias. |
| Backups mensais | 12 meses. |
| Metadados de certificado | Cinco anos ou prazo fiscal maior associado. |
| PFX substituído | Até 30 dias para reversão controlada. |
| PFX comprometido | Eliminação criptográfica depois do bloqueio e da preservação das evidências necessárias. |

Regras:

- seis anos será a política operacional conservadora inicial para o conjunto fiscal;
- prazo legal maior ou `legal hold` prevalecerá sobre a expiração normal;
- XML fiscal será mantido no formato exigido;
- dados reconstruídos no banco não substituirão a guarda do XML autorizado;
- payload não será mantido indefinidamente sem finalidade;
- descarte considerará cópias, réplicas e ciclo de backups;
- obrigações fiscais e de auditoria serão verificadas antes de eliminar ou anonimizar;
- a política será revisada por profissional jurídico ou fiscal antes da primeira produção e quando uma nova UF, regime ou cenário exigir regra diferente.

A eliminação do PFX removerá pacote cifrado e chave de dados protegida. Metadados e auditoria permanecerão pelo prazo definido, sem permitir recuperar a chave privada.

## Backup e recuperação

Requisitos mínimos:

- backups criptografados;
- acesso separado do acesso operacional comum;
- política de retenção e descarte;
- proteção contra exclusão acidental;
- restauração testada antes da produção;
- responsáveis e procedimento documentados;
- preservação da associação entre banco, XML e protocolo.

Objetivos iniciais:

- RPO do Postgres de até 15 minutos;
- RPO do XML autorizado próximo de zero depois de uma resposta de sucesso;
- RTO geral de até quatro horas;
- início da contenção de credencial crítica comprometida em até uma hora.

O sistema não considerará a autorização concluída para o cliente enquanto XML e protocolo não estiverem persistidos com segurança. Alertas e rotinas operacionais serão detalhados na Etapa 10. A estrutura física e a estratégia por schema serão definidas na Etapa 9.

## Resposta a incidentes

O plano mínimo deverá cobrir:

- API key exposta;
- certificado ou senha comprometidos;
- conta administrativa comprometida;
- acesso indevido entre grupos;
- vazamento de XML ou payload;
- alteração de artefato fiscal;
- comprometimento do banco ou armazenamento;
- perda ou corrupção de dados;
- segredo incluído em código, log ou build.

Fluxo mínimo:

```text
detectar
  -> conter
  -> preservar evidências
  -> revogar ou rotacionar credenciais
  -> avaliar impacto
  -> recuperar
  -> comunicar quando necessário
  -> corrigir a causa
  -> registrar aprendizados
```

Antes da produção deverão existir:

- responsável por segurança;
- responsável por privacidade e LGPD;
- responsável fiscal;
- administrador de credenciais;
- responsável por incidentes e seus substitutos;
- canal de emergência;
- procedimento de revogação;
- critério de severidade;
- preservação de evidências;
- critérios de comunicação ao grupo, titulares e ANPD;
- registro dos incidentes;
- ao menos uma simulação do procedimento.

Quando a plataforma confirmar incidente relevante em tratamento realizado como operadora, notificará o grupo imediatamente, com meta máxima de 24 horas. A comunicação inicial poderá ser preliminar e será complementada conforme a investigação.

A plataforma fornecerá ao grupo as informações necessárias para avaliar comunicação à ANPD e aos titulares. Quando atuar como controladora do tratamento afetado, assumirá diretamente essa avaliação e comunicação.

Uma mesma pessoa poderá exercer mais de uma responsabilidade em uma equipe pequena, desde que a atribuição, o substituto e a atuação em cada incidente sejam registrados.

### Severidade

| Severidade | Exemplo |
| --- | --- |
| Baixa | Tentativa bloqueada sem evidência de acesso. |
| Média | Credencial suspeita sem evidência de uso indevido. |
| Alta | Acesso indevido confirmado ou certificado exposto. |
| Crítica | Vazamento entre tenants, fraude fiscal ou comprometimento amplo. |

Procedimentos mínimos:

- API key exposta: revogar, gerar nova e investigar o histórico de uso;
- certificado comprometido: bloquear, comunicar a empresa, substituir e avaliar revogação na autoridade certificadora;
- acesso entre tenants: interromper o acesso, preservar evidências, identificar dados afetados e avaliar comunicações obrigatórias;
- segredo incluído no Git: revogar primeiro; remover do histórico não tornará o segredo novamente seguro;
- XML alterado: bloquear o acesso, conferir o hash, restaurar a cópia válida e investigar a origem.

## Desenvolvimento seguro

Requisitos para a implementação:

- revisão obrigatória de autenticação, autorização, multi-tenancy e criptografia;
- secret scanning no repositório e no pipeline;
- análise de dependências e imagens;
- validação estrita de entradas;
- consultas parametrizadas;
- respostas externas sem detalhes internos;
- limites contra consumo irrestrito de recursos;
- testes negativos e de abuso;
- configuração segura por padrão;
- inventário de endpoints e versões;
- documentação das exceções e riscos aceitos.

Uma avaliação independente de segurança será obrigatória antes do primeiro cliente em produção. O escopo mínimo incluirá API pública, endpoints administrativos, autenticação, autorização por objeto, isolamento multi-tenant, certificados, download de XML, mapeamentos, rate limit e SSRF.

Nenhuma vulnerabilidade crítica ou alta poderá permanecer aberta na entrada em produção. Vulnerabilidades médias exigirão correção ou aceitação formal de risco com responsável e prazo. A periodicidade posterior dependerá do risco, dos contratos e da operação.

## Requisitos obrigatórios para a primeira produção

1. PFX e senha protegidos em repouso e trânsito.
2. Chave mestra separada dos dados.
3. API keys fortes, não recuperáveis e revogáveis.
4. HTTPS obrigatório.
5. Banco e armazenamento sem exposição pública.
6. Ambientes e segredos separados.
7. Isolamento de tenant testado.
8. Autorização por grupo, empresa e recurso.
9. Logs sem segredos ou conteúdo fiscal completo.
10. Auditoria das operações críticas.
11. XML autorizado preservado e protegido.
12. Backup criptografado e restauração testada.
13. Plano básico de resposta a incidentes.
14. Verificação de dependências e segredos no desenvolvimento.
15. Revisão de segurança antes da ativação produtiva.

## Arquitetura preparada

O desenho não deverá impedir:

- troca do provedor de KMS ou cofre;
- rotação de chaves criptográficas;
- autenticação administrativa por provedor de identidade;
- OAuth 2.0 Client Credentials para integrações futuras;
- armazenamento imutável de auditoria e XML;
- centralização de eventos de segurança;
- expansão para outros módulos fiscais;
- políticas diferentes por ambiente e nível de risco.

## Evoluções futuras, não obrigatórias no primeiro MVP

- HSM dedicado;
- certificado A3;
- dupla aprovação para toda ação crítica;
- arquitetura zero trust completa;
- SIEM e SOC operando continuamente;
- certificação ISO 27001;
- múltiplas regiões ativas;
- recuperação instantânea por tenant;
- rotação totalmente automática de todos os segredos;
- pentest contínuo;
- builds reproduzíveis e assinatura completa da cadeia de software.

Esses controles poderão ser antecipados se houver exigência legal, contratual ou mudança relevante no risco.

## Catálogo mínimo de testes de segurança

### Autenticação e API keys

- chave ausente, inválida, revogada e de ambiente incorreto;
- segredo não recuperável após criação;
- revogação com efeito imediato;
- rotação planejada com expiração da chave anterior;
- rate limit aplicado sem afetar outro tenant.

### Autorização e multi-tenancy

- documento de outro grupo;
- empresa de outro grupo;
- integração de outro grupo;
- certificado de outra empresa;
- manipulação de `group_id` e `schema_name`;
- reutilização de conexão depois de transação de outro tenant;
- mensagem de fila com identificadores incompatíveis;
- endpoint administrativo chamado com API key de grupo.

### Certificado e criptografia

- PFX ou senha incorretos;
- certificado vencido, incompatível ou de outro CNPJ;
- conteúdo criptografado alterado;
- provedor criptográfico indisponível;
- identidade sem permissão de descriptografia;
- ausência de material sensível em log, tracing, fila e arquivo temporário.

### XML e dados

- alteração detectada pelo hash;
- download sem pertencimento ou permissão;
- URL expirada ou não reutilizável além da política;
- payload acima dos limites;
- dados produtivos ausentes em desenvolvimento;
- descarte conforme a política definida.

### Operação e recuperação

- restauração de backup;
- revogação emergencial de API key;
- bloqueio ou substituição de certificado;
- simulação de acesso indevido entre tenants;
- preservação de evidências e trilha de auditoria.

## Critérios para concluir a Etapa 8

A etapa somente poderá ser marcada como planejada e pronta para implementação quando existirem:

1. classificação e catálogo lógico dos dados;
2. modelo de ameaças revisado;
3. definição das responsabilidades de proteção de dados;
4. arquitetura do certificado A1;
5. política de criptografia e gestão de chaves;
6. ciclo de vida harmonizado das API keys;
7. matriz de permissões administrativas;
8. requisitos de isolamento multi-tenant;
9. política de logs, erros e auditoria;
10. política de retenção e descarte;
11. estratégia mínima de backup e recuperação;
12. plano de resposta a incidentes;
13. requisitos de desenvolvimento seguro;
14. catálogo de testes de segurança;
15. lista explícita de controles adiados;
16. política conservadora documentada e revisão profissional registrada como portão obrigatório de produção.

## Decisões consolidadas na Etapa 8

1. Produção usará KMS, Vault ou equivalente, sem acoplamento a um fornecedor nesta etapa.
2. O PFX cifrado ficará no Postgres; XMLs ficarão em armazenamento de objetos privado.
3. Certificados usarão criptografia envelope com AES-256-GCM e uma chave de dados por versão.
4. A administração começará com cinco perfis de acesso separados.
5. A rotação de API keys terá 72 horas por padrão e limite máximo de sete dias.
6. A política inicial de retenção seguirá a tabela deste documento.
7. Responsabilidades de segurança, privacidade, fiscal, credenciais e incidentes deverão ter titulares e substitutos registrados antes da produção.
8. O objetivo inicial será RPO de 15 minutos para o banco, RPO próximo de zero para XML confirmado e RTO geral de quatro horas.
9. Haverá avaliação independente antes do primeiro cliente em produção, sem vulnerabilidade crítica ou alta aberta.
10. O grupo será, em regra, controlador; a plataforma, operadora; e fornecedores aplicáveis, suboperadores, sujeito à validação contratual.

## Validações obrigatórias antes da produção

A Etapa 8 está concluída para fins de planejamento. Antes de ativar o primeiro cliente em produção, o projeto deverá obter e registrar:

1. revisão jurídica dos papéis de controlador, operador e suboperador e das cláusulas de incidente e encerramento;
2. revisão fiscal ou contábil da retenção de seis anos para a primeira UF, regime e cenários do MVP;
3. confirmação do contrato ou termo de tratamento de dados;
4. identificação nominal dos responsáveis por segurança, privacidade, credenciais e incidentes;
5. avaliação independente de segurança prevista neste documento.

A ausência dessas evidências bloqueará produção, mas não a implementação. Como o projeto é conduzido inicialmente por uma única pessoa, os mesmos nomes poderão ocupar mais de uma responsabilidade, desde que cada papel e seus procedimentos sejam registrados.

## Resultado esperado

Ao concluir esta etapa, a plataforma terá requisitos de segurança suficientes para orientar uma implementação produtiva: certificados e segredos protegidos, isolamento multi-tenant verificável, acesso administrativo restrito, artefatos fiscais íntegros, logs seguros, auditoria, recuperação e resposta a incidentes.

A arquitetura estará preparada para aumentar sua maturidade sem exigir antecipadamente infraestrutura e processos desproporcionais ao primeiro MVP.
