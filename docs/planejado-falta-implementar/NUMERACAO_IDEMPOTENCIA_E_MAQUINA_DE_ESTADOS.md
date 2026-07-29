# Numeracao, idempotencia e maquina de estados

## Objetivo

Definir como a plataforma controla o ciclo de vida de um documento fiscal, impede emissoes duplicadas, reserva numeracao com seguranca e registra todas as tentativas de processamento.

Estas regras sao compartilhadas pela plataforma, mas cada modulo fiscal pode possuir particularidades de numeracao, transmissao, consulta e eventos posteriores.

## Principios

- O status apresentado ao cliente deve ser simples e estavel.
- O processamento interno deve possuir estados detalhados para operacao e auditoria.
- O documento fiscal e suas tentativas de emissao sao registros diferentes.
- Toda solicitacao de emissao deve ser idempotente.
- Numeros fiscais sao reservados de forma atomica e nunca reutilizados automaticamente.
- Uma falha de comunicacao nao significa que a autoridade fiscal deixou de processar o documento.
- Somente erros temporarios podem gerar retentativa automatica.
- Toda transicao de estado deve gerar historico imutavel.

## Status publico e estado interno

A API externa deve expor um conjunto pequeno de status:

```text
pending
processing
authorized
rejected
cancelled
failed
```

Os detalhes operacionais ficam em estados internos e codigos de motivo. Isso permite evoluir o processamento sem alterar constantemente o contrato publico.

### Mapeamento inicial

| Estado interno | Status publico | Significado |
| --- | --- | --- |
| `received` | `pending` | Solicitacao aceita e registrada. |
| `normalizing` | `pending` | Dados sendo mapeados e normalizados. |
| `validating` | `pending` | Modelo interno em validacao. |
| `ready` | `pending` | Documento valido e pronto para emissao. |
| `number_reserved` | `pending` | Numeracao fiscal reservada pelo modulo. |
| `queued` | `pending` | Documento aguardando worker. |
| `processing` | `processing` | Tentativa de emissao em andamento. |
| `retry_scheduled` | `processing` | Nova tentativa temporaria agendada. |
| `authorization_unknown` | `processing` | Transmissao ocorreu, mas o resultado ainda precisa ser conciliado. |
| `reconciling` | `processing` | Plataforma consultando a situacao fiscal antes de retransmitir. |
| `authorized` | `authorized` | Documento autorizado pela autoridade fiscal. |
| `rejected` | `rejected` | Autoridade fiscal recusou o documento. |
| `validation_failed` | `failed` | Documento nao passou na validacao interna. |
| `processing_failed` | `failed` | Processamento terminou sem recuperacao automatica. |
| `cancelled` | `cancelled` | Evento de cancelamento autorizado. |

Um rascunho incompleto da interface nao faz parte do ciclo de emissao. O documento fiscal entra nessa maquina somente quando o usuario ou integracao solicita seu processamento.

## Fluxo principal

```text
received
  -> normalizing
  -> validating
      -> validation_failed
      -> ready
          -> number_reserved
          -> queued
          -> processing
              -> authorized
              -> rejected
              -> retry_scheduled -> queued
              -> authorization_unknown -> reconciling
              -> processing_failed
```

A reconciliacao pode terminar em `authorized`, `rejected`, `retry_scheduled` ou `processing_failed`, conforme a resposta obtida e a regra do modulo fiscal.

Transicoes fora do fluxo definido devem ser recusadas. A alteracao de estado deve ocorrer junto da gravacao do respectivo evento, evitando documento atualizado sem historico.

## Numeracao fiscal

### Escopo

A politica de numeracao pertence ao modulo fiscal. Quando o documento exigir controle sequencial, o contador deve considerar os elementos aplicaveis, como:

```text
empresa + ambiente + tipo/modelo + serie -> proximo numero
```

Outras dimensoes somente podem ser adicionadas quando forem exigidas pelo documento ou pela autoridade fiscal correspondente.

### Momento da reserva

O numero deve ser reservado depois que o documento estiver normalizado e validado, imediatamente antes de entrar no fluxo de emissao.

Isso evita consumir numeracao para solicitacoes que ja seriam recusadas pela validacao interna e garante que o XML seja produzido com uma identidade fiscal previamente reservada.

### Concorrencia

A reserva deve ser atomica e transacional. Duas requisicoes concorrentes da mesma empresa, ambiente, modelo e serie nunca podem receber o mesmo numero.

O mecanismo fisico de bloqueio e persistencia sera definido na etapa de banco de dados, mas o banco deve ser a autoridade final do contador.

### Numero reservado

Depois de reservado, um numero nao pode ser devolvido ao contador nem reutilizado automaticamente.

Em caso de falha, o modulo deve decidir entre consultar a autoridade fiscal, realizar nova tentativa com a mesma identidade fiscal ou iniciar o procedimento fiscal aplicavel. A decisao precisa ser registrada e nunca pode ser tomada apenas porque houve timeout.

## Idempotencia

### Obrigatoriedade

Toda solicitacao externa de emissao deve possuir uma chave de idempotencia:

```http
Idempotency-Key: pedido-123-nfe
```

A interface oficial da plataforma deve gerar e enviar essa chave. Integracoes diretas devem cria-la a partir de um identificador estavel do processo de origem.

### Escopo da chave

A unicidade deve ser avaliada dentro do contexto autenticado:

```text
grupo + empresa + integracao + tipo de documento + operacao + idempotency_key
```

Quando nao houver integracao, o campo correspondente usa um contexto interno conhecido, sem alterar os demais componentes do escopo.

### Impressao digital da solicitacao

Na primeira chamada, a plataforma deve gerar e armazenar um `request_fingerprint` a partir do conteudo canonico normalizado usado para criar o documento.

O processo de canonicalizacao deve garantir que diferencas irrelevantes, como ordem das propriedades JSON, nao alterem o hash. Versoes de contrato e mapeamento que influenciem o resultado tambem devem ser registradas.

### Comportamento

| Situacao | Resultado |
| --- | --- |
| Chave nova | Cria a solicitacao e o documento fiscal. |
| Mesma chave e mesma impressao digital | Retorna o documento ja existente. |
| Mesma chave com documento em processamento | Retorna o identificador e o status atual. |
| Mesma chave com dados diferentes | Retorna conflito `409`. |

Uma retentativa idempotente nunca cria um novo documento, reserva outro numero ou inicia processamento paralelo para a mesma solicitacao.

A politica de retencao das chaves sera definida junto da politica de retencao fiscal. Enquanto o documento existir e puder ser consultado ou reprocessado, sua chave nao pode ser reutilizada.

## Documento e tentativas

O documento fiscal representa a intencao e o resultado consolidado da emissao. Cada execucao tecnica deve ser registrada separadamente.

```text
fiscal_document
  -> issuance_attempt 1
  -> issuance_attempt 2
  -> reconciliation_attempt 1
```

Cada tentativa deve registrar, no minimo:

- identificador e tipo da tentativa;
- documento fiscal relacionado;
- numero sequencial da tentativa;
- estado inicial e final;
- data de inicio e termino;
- worker responsavel;
- versao do modulo e do layout;
- identificador ou hash do XML transmitido;
- codigo e mensagem de retorno;
- protocolo, recibo ou referencia externa quando houver;
- categoria do erro;
- data da proxima tentativa, quando agendada.

Uma nova tentativa nunca sobrescreve a anterior.

## Resultado de autorizacao desconhecido

Quando a transmissao for iniciada, mas a resposta nao puder ser confirmada, o documento deve entrar em `authorization_unknown`.

Exemplos:

- timeout depois do envio;
- conexao encerrada durante a resposta;
- worker interrompido depois da transmissao;
- resposta externa incompleta ou nao persistida.

Nesse estado, a plataforma deve executar reconciliacao usando os mecanismos permitidos pelo modulo fiscal. Uma retransmissao somente pode ocorrer depois que a reconciliacao concluir que ela e segura.

## Politica de retentativas

### Erros temporarios

Podem gerar retentativa automatica:

- timeout antes da confirmacao do resultado;
- indisponibilidade temporaria da autoridade fiscal;
- falha transitoria de rede;
- erro temporario de infraestrutura;
- interrupcao recuperavel do worker.

### Erros nao retentaveis automaticamente

- payload ou modelo interno invalido;
- regra fiscal invalida ou nao suportada;
- rejeicao da autoridade fiscal;
- certificado invalido, vencido ou nao autorizado;
- configuracao fiscal ausente;
- operacao ou modulo nao habilitado.

As retentativas devem possuir limite, intervalo progressivo e variacao aleatoria para evitar concentracao de chamadas. Valores exatos podem ser configurados por modulo e categoria de erro.

Ao atingir o limite, o documento deve ir para `processing_failed` e exigir decisao operacional explicita. Uma acao manual de reprocessamento cria nova tentativa e preserva todo o historico anterior.

## Concorrencia entre workers

Um documento so pode possuir uma tentativa ativa por vez. O worker deve adquirir uma concessao temporaria de processamento, com identificador, inicio e expiracao.

Se o worker for interrompido, outro worker pode assumir somente depois da expiracao ou liberacao controlada da concessao. Antes de retransmitir, ele deve verificar se a tentativa anterior pode ter alcançado a autoridade fiscal.

O desenho fisico da fila, bloqueios e concessoes sera definido nas etapas de banco e implementacao.

## Eventos posteriores

Cancelamento e outros eventos fiscais nao devem apagar ou reabrir a emissao original. Eles devem possuir fluxo e tentativas proprias vinculadas ao documento autorizado.

Um cancelamento somente altera o status publico para `cancelled` depois da confirmacao da autoridade fiscal. Falha ou rejeicao do cancelamento preserva o documento como `authorized` e registra o evento correspondente.

## Historico de transicoes

Toda mudanca de estado deve gerar um registro imutavel:

```text
document_id
previous_state
new_state
reason_code
actor_type
actor_id
attempt_id
occurred_at
metadata
```

O ator pode ser API, usuario, worker, administrador ou autoridade fiscal. Correcoes operacionais devem gerar novos eventos; eventos existentes nao podem ser editados ou excluidos.

## Resultado da etapa

Esta etapa estabelece:

- status publico separado do estado interno;
- fluxo permitido de estados e transicoes;
- numeracao atomica e controlada por modulo;
- idempotencia obrigatoria e conflito para conteudo diferente;
- separacao entre documento e tentativas;
- reconciliacao obrigatoria para resultado desconhecido;
- retentativas somente para erros temporarios;
- concorrencia controlada entre workers;
- historico imutavel de transicoes.
