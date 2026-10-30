# Trigger.dev self-host no Coolify

Stack único (webapp + worker) para um servidor Docker. Imagens `v4.5.0`. Não uses o compose de desenvolvimento do repositório (`triggerdotdev-docker`): esse é o ambiente de quem contribui para o Trigger.dev.

Ficheiro: `docker-compose.coolify.yml`, ao lado dos XML em `clickhouse/`.

## Recurso no Coolify

Git, não "Docker Compose Empty". Os XML do ClickHouse são ficheiros do repositório.

| Campo | Valor |
| --- | --- |
| Repositório | `DoutorVida/trigger.dev` |
| Base directory | `/hosting/docker` |
| Compose file | `docker-compose.coolify.yml` |
| Preserve repository during deployment | ligado |

Um stack por servidor. As redes chamam-se `webapp`, `supervisor` e `docker-proxy`, e o supervisor liga os runs a `webapp` e `supervisor`.

Máquina única: pelo menos 8 vCPU e 16 GB de RAM. O contentor `docker-proxy` precisa de `/var/run/docker.sock` no servidor.

## Segredos

O Coolify gera cada `SERVICE_HEX_32_*` (32 caracteres hex). Esse tamanho é o que o `ENCRYPTION_KEY` exige (32 bytes). Não os edites e não definas `TRIGGER_WORKER_TOKEN`: o webapp grava o token no volume `shared` e o supervisor lê-o.

| Variável | Uso |
| --- | --- |
| `SERVICE_HEX_32_POSTGRES` | Password do Postgres |
| `SERVICE_HEX_32_SESSION` | `SESSION_SECRET` |
| `SERVICE_HEX_32_MAGICLINK` | `MAGIC_LINK_SECRET` |
| `SERVICE_HEX_32_ENCRYPTION` | `ENCRYPTION_KEY` |
| `SERVICE_HEX_32_PROVIDER` | `PROVIDER_SECRET` |
| `SERVICE_HEX_32_COORDINATOR` | `COORDINATOR_SECRET` |
| `SERVICE_HEX_32_MANAGEDWORKER` | Segredo partilhado webapp/supervisor |
| `SERVICE_HEX_32_CLICKHOUSE` | Password do ClickHouse |
| `SERVICE_HEX_32_OBJECTSTORE` | Password do MinIO |
| `SERVICE_HEX_32_REGISTRY` | Password do registry (`registry-user`) |

## Domínios

O `:3000` e o `:5000` são a porta dentro do contentor. O URL público fica na 443.

| Serviço | Domínio no Coolify | Público |
| --- | --- | --- |
| webapp | `https://trigger4.<dominio>:3000` | `https://trigger4.<dominio>` |
| registry | `https://registry.trigger4.<dominio>:5000` | `https://registry.trigger4.<dominio>` |

Em produção troca `trigger4` por `trigger3`. Staging e produção não partilham este stack.

Depois de gravar, confirma estas. Corrige se o Coolify deixar barra no fim ou a porta no URL:

| Variável | Valor |
| --- | --- |
| `APP_ORIGIN` | `https://trigger4.<dominio>` |
| `LOGIN_ORIGIN` | o mesmo |
| `API_ORIGIN` | o mesmo |
| `DOCKER_REGISTRY_URL` | `registry.trigger4.<dominio>` (sem `https://` e sem porta). Enquanto estiver vazio, o stack usa `registry:5000` |
| `DOCKER_REGISTRY_USERNAME` | `registry-user` |
| `TRIGGER_IMAGE_TAG` | `v4.5.0` |
| `NODE_MAX_OLD_SPACE_SIZE` | `4800` com 6 GB para o webapp; `6400` com 8 GB |

Limites de staging, já no compose (substituem os defaults da imagem: payload 3 MB, batch 1 MB, metadata 256 KB, concorrência de ambiente 100):

| Variável | Valor |
| --- | --- |
| `TASK_PAYLOAD_MAXIMUM_SIZE` | 10 MB |
| `BATCH_TASK_PAYLOAD_MAXIMUM_SIZE` | 10 MB |
| `TASK_RUN_METADATA_MAXIMUM_SIZE` | 1 MB |
| `DEFAULT_ORG_EXECUTION_CONCURRENCY_LIMIT` | `1000` (gravado na organização na criação) |
| `DEFAULT_ENV_EXECUTION_CONCURRENCY_LIMIT` | `1000` |
| `RUN_ENGINE_WORKER_CONCURRENCY_LIMIT` | `50` |
| `EVENT_REPOSITORY_DEFAULT_STORE` | `clickhouse_v2` |
| `OBJECT_STORE_ACCESS_KEY_ID` | `admin` |

## Email

Não definas `EMAIL_TRANSPORT` vazio: a aplicação recusa a string vazia e o webapp reinicia em ciclo. Enquanto a variável não existir, o magic link aparece nos logs do contentor `webapp`.

Quando o SMTP estiver acessível a partir do servidor:

| Variável | Valor |
| --- | --- |
| `EMAIL_TRANSPORT` | `smtp` |
| `SMTP_HOST` | `mail.doutorvida.pt` |
| `SMTP_PORT` | `587` |
| `SMTP_SECURE` | `false` |
| `SMTP_USER` | conta SMTP |
| `SMTP_PASSWORD` | password SMTP |
| `FROM_EMAIL` | `admin@doutorvida.pt` |
| `REPLY_TO_EMAIL` | `admin@doutorvida.pt` |
| `WHITELISTED_EMAILS` | `.*@doutorvida\.pt$` |
| `ADMIN_EMAILS` | `admin@doutorvida\.pt` |

`ADMIN_EMAILS` e `WHITELISTED_EMAILS` são regex. `RESEND_API_KEY`, `AUTH_GITHUB_CLIENT_ID` e `AUTH_GITHUB_CLIENT_SECRET` ficam por definir se usares SMTP.

## Registry e deploy de tasks

Na máquina que corre `trigger deploy`, e no próprio servidor:

```bash
docker login registry.trigger4.<dominio>
```

User `registry-user`. Password: o valor de `SERVICE_HEX_32_REGISTRY`.

No primeiro arranque o registry pode falhar o pull a partir do próprio servidor (o Docker a falar com o domínio público). Se isso acontecer, no servidor do Coolify aponta o hostname do registry para o IP do proxy.

## Nest (outro recurso Coolify)

| Variável | Valor |
| --- | --- |
| `TRIGGER_API_URL` | `https://trigger4.<dominio>` |
| `WORKFLOWS_DASHBOARD_EMBED_URL` | o mesmo |
| `TRIGGER_SECRET_KEY` | secret key do projecto, criada no dashboard depois do login |
| `WORKFLOW_ENGINE_ENABLED` | `true` |
| `TRIGGER_DV_SERVICE_NAME` | `dv-trigger-worker` |
| `TRIGGER_DV_SERVICE_SECRET` | secret do service account |
| `DV_INTERNAL_WORKFLOW_BASE_URL` | URL interna do Nest, alcançável pelos contentores dos runs |
