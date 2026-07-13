# Autenticação e gerenciamento de API keys

## Decisão para o MVP

A API será consumida por sistemas de clientes, como ERPs e e-commerces. Por isso, a autenticação inicial será máquina a máquina, usando uma **API key** por cliente.

Não haverá login de usuário para o ERP consumir a API. Login de usuário será uma responsabilidade futura do painel administrativo, usado pela equipe interna para cadastrar clientes e administrar suas chaves.

O cliente envia a chave em todas as chamadas:

```http
Authorization: Bearer nf_live_key_01HZX...<segredo>
```

A chave identifica o cliente. A API usa essa identidade para decidir quais empresas, notas e operações ele pode acessar.

O formato recomendado é:

```text
nf_live_<key_id>.<segredo>
```

- `nf_live` identifica o ambiente de produção. Chaves de teste usam `nf_test`.
- `key_id` é um identificador público da credencial, usado para localizar seu registro rapidamente.
- `segredo` é a parte confidencial, longa e aleatória. Ela não pode ser recuperada depois da criação.

## Uma chave ativa por cliente

No MVP, cada cliente terá apenas uma API key ativa. Isso simplifica a operação e o suporte.

Quando a chave for resetada, todas as chaves ativas anteriores do cliente são revogadas imediatamente. Apenas a nova chave continuará com acesso à API. Isso significa que o ERP ficará sem acesso até que o cliente cadastre a nova chave; esse é o comportamento intencional para perda ou suspeita de vazamento.

No futuro, se um mesmo cliente precisar integrar mais de um sistema, a regra poderá evoluir para uma chave por integração. Nesse cenário, o reset deverá ser feito para uma integração específica, sem interromper as demais.

## Ciclo de vida da chave

```text
Criação inicial
  -> active
  -> revoked (revogação ou reset)
  -> deleted (exclusão lógica, somente após revogação)
```

Uma chave não pode ser recuperada ou reenviada. A API grava apenas um hash do segredo. Se o cliente perder a chave, é necessário gerar uma nova por meio do reset.

## Endpoints administrativos

Estes endpoints são internos. Eles não podem ser chamados usando a API key do cliente. Devem exigir autenticação do painel administrativo, permissões administrativas e, quando houver painel, MFA.

Uma API key de cliente permite apenas consumir os endpoints externos autorizados. Ela nunca permite criar, listar, revogar, excluir ou resetar chaves.

| Ação | Método e rota | Regra |
| --- | --- | --- |
| Criar primeira chave | `POST /admin/v1/clients/{client_id}/api-keys` | Só cria se o cliente não possuir chave ativa. |
| Listar chaves | `GET /admin/v1/clients/{client_id}/api-keys` | Retorna metadados; nunca retorna o segredo. |
| Revogar chave | `POST /admin/v1/api-keys/{api_key_id}/revoke` | A chave deixa de funcionar imediatamente. |
| Excluir chave | `DELETE /admin/v1/api-keys/{api_key_id}` | Permitido apenas para chave já revogada; realiza exclusão lógica. |
| Resetar chave | `POST /admin/v1/clients/{client_id}/api-keys/reset` | Revoga as chaves ativas do cliente e cria uma nova. |

## Criação inicial

Exemplo:

```http
POST /admin/v1/clients/jorge/api-keys
```

Resposta:

```json
{
  "id": "key_01",
  "api_key": "nf_live_chave_secreta_aleatoria",
  "status": "active"
}
```

O campo `api_key` completo é retornado somente nesta resposta. Depois disso, ele não poderá ser consultado novamente.

## Reset de chave

O reset atende casos como perda da chave, troca preventiva ou suspeita de vazamento.

```http
POST /admin/v1/clients/jorge/api-keys/reset
```

Resposta:

```json
{
  "id": "key_02",
  "api_key": "nf_live_nova_chave_secreta_aleatoria",
  "status": "active",
  "revoked_keys_count": 1
}
```

O reset deve ocorrer em uma única transação de banco de dados:

1. Localizar e bloquear as chaves ativas do cliente.
2. Revogar todas as chaves ativas encontradas.
3. Gerar a nova chave aleatória.
4. Salvar apenas o hash da nova chave.
5. Registrar a auditoria do reset.
6. Confirmar a transação e devolver o segredo uma única vez.

Se alguma etapa falhar, a transação deve ser desfeita e a chave anterior deve continuar válida.

Se a aplicação usar cache para validar credenciais, o cache da chave revogada deve ser invalidado antes de retornar sucesso. A chave antiga não pode continuar válida até o fim de um TTL de cache.

## Listagem de chaves

Exemplo de retorno de `GET /admin/v1/clients/{client_id}/api-keys`:

```json
[
  {
    "id": "key_01",
    "prefix": "nf_live_ab12",
    "status": "revoked",
    "created_at": "2026-06-01T10:00:00Z",
    "revoked_at": "2026-07-13T10:00:00Z",
    "last_used_at": "2026-07-12T16:42:00Z"
  },
  {
    "id": "key_02",
    "prefix": "nf_live_cd34",
    "status": "active",
    "created_at": "2026-07-13T10:00:00Z",
    "last_used_at": null
  }
]
```

O prefixo ajuda a identificar a chave em suporte e auditoria sem expor seu segredo.

## Modelo de dados sugerido

```text
api_keys
- id
- client_id
- prefix
- secret_hash
- status                    -- active | revoked | deleted | expired
- created_at
- last_used_at
- revoked_at
- revoked_by_admin_id
- revoke_reason
- deleted_at
- deleted_by_admin_id
```

### Regra de integridade no banco

O banco de dados, e não apenas a aplicação, deve garantir que um cliente tenha no máximo uma chave ativa. Em Postgres, a regra pode ser implementada com um índice único parcial:

```sql
CREATE UNIQUE INDEX api_keys_one_active_per_client
ON api_keys (client_id)
WHERE status = 'active';
```

Essa proteção evita duas chaves ativas caso duas solicitações de criação ou reset ocorram ao mesmo tempo.

## Validação de uma chamada externa

Para toda chamada de cliente, a API deve executar:

```text
chave recebida
  -> localizar pelo prefixo
  -> validar o segredo contra o hash armazenado
  -> confirmar que o status é active
  -> identificar o cliente dono da chave
  -> verificar se a operação é permitida
  -> verificar se a empresa e a nota pertencem ao cliente
  -> permitir ou negar a chamada
```

Uma chave válida não basta para acessar qualquer recurso. A API precisa sempre verificar se a nota ou empresa solicitada pertence ao cliente autenticado.

Respostas de autenticação e autorização devem seguir esta regra:

- `401 Unauthorized`: chave ausente, inválida, revogada, expirada ou pertencente ao ambiente errado.
- `403 Forbidden`: chave válida, mas sem permissão para a empresa, nota ou operação solicitada.

Quando necessário para evitar revelar a existência de um recurso de outro cliente, a API poderá responder `404 Not Found` após aplicar o filtro de pertencimento.

## Requisitos de segurança obrigatórios

- Gerar o segredo usando `crypto/rand` do Go, com ao menos 32 bytes aleatórios.
- Armazenar somente um hash/HMAC do segredo, nunca seu valor completo.
- Comparar o segredo recebido com o valor armazenado em tempo constante.
- Exibir o segredo completo apenas na criação ou no reset.
- Usar HTTPS em todos os ambientes acessíveis externamente.
- Fazer a chave revogada falhar imediatamente com `401 Unauthorized`.
- Nunca registrar API keys, senhas de certificado ou XML fiscal em logs.
- Aplicar rate limit e quota de uso por API key e por IP.
- Manter auditoria de criação, uso, revogação, reset e exclusão, incluindo administrador responsável, data, motivo e IP de origem.
- Separar chaves de teste e produção; uma chave não pode funcionar nos dois ambientes.
- Fazer exclusão lógica para preservar rastreabilidade e investigação de incidentes.
- Proteger os endpoints administrativos com login, permissão específica de gerenciamento de chaves e MFA.

## Evolução futura

Se o produto passar a atender integrações corporativas mais complexas, a autenticação poderá evoluir para OAuth 2.0 Client Credentials com tokens JWT de curta duração. Essa evolução não substitui a necessidade de autorização por cliente, empresa e recurso.

Para o MVP, API key por cliente com reset imediato é a abordagem escolhida.
