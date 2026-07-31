# Autenticação e gerenciamento de API keys

## Decisão para o MVP

A API será consumida por sistemas dos grupos, como ERPs e e-commerces. Por isso, a autenticação inicial será máquina a máquina, usando **API keys** vinculadas ao grupo e ao ambiente.

Não haverá login de usuário para o ERP consumir a API. Login de usuário será uma responsabilidade futura do painel administrativo, usado pela equipe interna para cadastrar grupos e administrar suas chaves.

O sistema do grupo envia a chave em todas as chamadas:

```http
Authorization: Bearer nf_live_key_01HZX...<segredo>
```

A chave identifica o grupo. A API usa essa identidade para decidir quais empresas, documentos e operações ele pode acessar.

O formato recomendado é:

```text
nf_live_<key_id>.<segredo>
```

- `nf_live` identifica o ambiente de produção. Chaves de teste usam `nf_test`.
- `key_id` é um identificador público da credencial, usado para localizar seu registro rapidamente.
- `segredo` é a parte confidencial, longa e aleatória. Ela não pode ser recuperada depois da criação.

## Chave principal e janela de rotação

No funcionamento normal, cada grupo terá uma chave principal ativa por ambiente.

Durante uma rotação planejada, uma segunda chave poderá permanecer ativa junto com a anterior. A janela padrão será de 72 horas e terá limite máximo de sete dias. Ao final da janela, a chave anterior será revogada automaticamente.

Esse período permite atualizar o ERP sem interromper o serviço, mas não poderá ser usado para manter indefinidamente várias credenciais ativas.

Quando houver perda ou suspeita de vazamento, será usado o reset emergencial: todas as chaves ativas anteriores serão revogadas imediatamente e somente a nova continuará válida.

No futuro, se um mesmo grupo precisar integrar mais de um sistema com credenciais independentes, a regra poderá evoluir para uma chave por integração. Nesse cenário, o reset deverá ser feito para uma integração específica, sem interromper as demais.

## Ciclo de vida da chave

```text
Criação inicial
  -> active
  -> active durante rotação, com prazo de expiração
  -> revoked (revogação ou reset)
  -> expired (fim automático da janela de rotação)
  -> deleted (exclusão lógica, somente após revogação)
```

Uma chave não pode ser recuperada ou reenviada. A API grava apenas um hash ou HMAC do segredo. Se o grupo perder a chave, é necessário gerar uma nova por meio do reset.

## Endpoints administrativos

Estes endpoints são internos. Eles não podem ser chamados usando a API key do grupo. Devem exigir autenticação do painel administrativo, permissões administrativas e, quando houver painel, MFA.

Uma API key de grupo permite apenas consumir os endpoints externos autorizados. Ela nunca permite criar, listar, rotacionar, revogar, excluir ou resetar chaves.

| Ação | Método e rota | Regra |
| --- | --- | --- |
| Criar primeira chave | `POST /admin/v1/groups/{group_id}/api-keys` | Só cria se o grupo ainda não possuir chave. |
| Iniciar rotação | `POST /admin/v1/groups/{group_id}/api-keys/rotate` | Cria a nova chave e agenda a expiração da anterior. |
| Listar chaves | `GET /admin/v1/groups/{group_id}/api-keys` | Retorna metadados; nunca retorna o segredo. |
| Revogar chave | `POST /admin/v1/api-keys/{api_key_id}/revoke` | A chave deixa de funcionar imediatamente. |
| Excluir chave | `DELETE /admin/v1/api-keys/{api_key_id}` | Permitido apenas para chave já revogada; realiza exclusão lógica. |
| Resetar chave | `POST /admin/v1/groups/{group_id}/api-keys/reset` | Revoga as chaves ativas do grupo e cria uma nova. |

## Criação inicial

Exemplo:

```http
POST /admin/v1/groups/group_01/api-keys
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

## Rotação planejada

A rotação atende trocas preventivas sem indisponibilidade do ERP.

```http
POST /admin/v1/groups/group_01/api-keys/rotate
```

Resposta:

```json
{
  "id": "key_02",
  "api_key": "nf_live_nova_chave_secreta_aleatoria",
  "status": "active",
  "previous_key_expires_at": "2026-08-03T10:00:00Z"
}
```

A operação deverá:

1. bloquear o conjunto de chaves do grupo e ambiente;
2. recusar a operação se já existir rotação em andamento;
3. gerar a nova chave;
4. salvar somente o hash ou HMAC do novo segredo;
5. agendar a expiração da chave anterior em 72 horas, respeitando o limite máximo de sete dias;
6. registrar a auditoria;
7. devolver o novo segredo uma única vez.

Se a nova chave for confirmada antes do prazo, a chave anterior poderá ser revogada imediatamente. Se nenhuma confirmação ocorrer, ela será revogada automaticamente no prazo registrado.

## Reset de chave

O reset atende casos como perda da chave, troca preventiva ou suspeita de vazamento.

```http
POST /admin/v1/groups/group_01/api-keys/reset
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

1. Localizar e bloquear as chaves ativas do grupo e ambiente.
2. Revogar todas as chaves ativas encontradas.
3. Gerar a nova chave aleatória.
4. Salvar apenas o hash da nova chave.
5. Registrar a auditoria do reset.
6. Confirmar a transação e devolver o segredo uma única vez.

Se alguma etapa falhar, a transação deve ser desfeita e a chave anterior deve continuar válida.

Se a aplicação usar cache para validar credenciais, o cache da chave revogada deve ser invalidado antes de retornar sucesso. A chave antiga não pode continuar válida até o fim de um TTL de cache.

## Listagem de chaves

Exemplo de retorno de `GET /admin/v1/groups/{group_id}/api-keys`:

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
- group_id
- environment
- prefix
- secret_hash
- status                    -- active | revoked | deleted | expired
- replaces_key_id
- rotation_expires_at
- created_at
- last_used_at
- revoked_at
- revoked_by_admin_id
- revoke_reason
- deleted_at
- deleted_by_admin_id
```

### Regra de integridade no banco

O banco de dados e a transação de rotação deverão garantir, por grupo e ambiente:

- uma chave ativa fora de rotação;
- no máximo duas chaves ativas durante a janela;
- somente uma rotação em andamento;
- expiração obrigatória da chave anterior;
- impossibilidade de uma chave funcionar em outro ambiente.

O mecanismo físico de restrições, bloqueios e expiração será definido na etapa de banco de dados e migrations.

## Validação de uma chamada externa

Para toda chamada externa, a API deve executar:

```text
chave recebida
  -> localizar pelo prefixo
  -> validar o segredo contra o hash armazenado
  -> confirmar que o status é active
  -> identificar o grupo dono da chave
  -> verificar se a operação é permitida
  -> verificar se a empresa e o documento pertencem ao grupo
  -> permitir ou negar a chamada
```

Uma chave válida não basta para acessar qualquer recurso. A API precisa sempre verificar se o documento ou empresa solicitada pertence ao grupo autenticado.

Respostas de autenticação e autorização devem seguir esta regra:

- `401 Unauthorized`: chave ausente, inválida, revogada, expirada ou pertencente ao ambiente errado.
- `403 Forbidden`: chave válida, mas sem permissão para a empresa, nota ou operação solicitada.

Quando necessário para evitar revelar a existência de um recurso de outro grupo, a API poderá responder `404 Not Found` após aplicar o filtro de pertencimento.

## Requisitos de segurança obrigatórios

- Gerar o segredo usando `crypto/rand` do Go, com ao menos 32 bytes aleatórios.
- Armazenar somente um hash/HMAC do segredo, nunca seu valor completo.
- Comparar o segredo recebido com o valor armazenado em tempo constante.
- Exibir o segredo completo apenas na criação, rotação ou reset.
- Expirar automaticamente a chave anterior ao final da janela de rotação.
- Usar HTTPS em todos os ambientes acessíveis externamente.
- Fazer a chave revogada falhar imediatamente com `401 Unauthorized`.
- Nunca registrar API keys, senhas de certificado ou XML fiscal em logs.
- Aplicar rate limit e quota de uso por API key e por IP.
- Manter auditoria de criação, uso, revogação, reset e exclusão, incluindo administrador responsável, data, motivo e IP de origem.
- Separar chaves de teste e produção; uma chave não pode funcionar nos dois ambientes.
- Fazer exclusão lógica para preservar rastreabilidade e investigação de incidentes.
- Proteger os endpoints administrativos com login, permissão específica de gerenciamento de chaves e MFA.

## Evolução futura

Se o produto passar a atender integrações corporativas mais complexas, a autenticação poderá evoluir para OAuth 2.0 Client Credentials com tokens JWT de curta duração. Essa evolução não substitui a necessidade de autorização por grupo, empresa e recurso.

Para o MVP, API key por grupo e ambiente, com rotação planejada e reset emergencial imediato, é a abordagem escolhida.
