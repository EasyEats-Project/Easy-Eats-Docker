# Easy Eats Docker

Infraestrutura Docker do EasyEats: PostgreSQL, backend (Java 21 / Spring Boot) e frontend (Angular servido por Nginx), iniciados com um único comando. O app mobile (Expo) roda fora do Docker.

Convenções deste guia:

- Todos os comandos `docker compose` são executados dentro de `infra/` (`cd infra`).
- Os comandos `git` e `Copy-Item` são executados na raiz do repositório.
- Os exemplos de shell usam PowerShell (Windows).

## Sumário

1. [Pré-requisitos](#1-pré-requisitos)
2. [Organização dos repositórios](#2-organização-dos-repositórios)
3. [Docker Desktop e WSL2](#3-docker-desktop-e-wsl2)
4. [Configuração do `.env`](#4-configuração-do-env)
5. [Inicialização](#5-inicialização)
6. [Encerramento](#6-encerramento)
7. [Logs](#7-logs)
8. [Reinício de serviços](#8-reinício-de-serviços)
9. [Persistência do banco](#9-persistência-do-banco)
10. [Mobile (Expo) fora do Docker](#10-mobile-expo-fora-do-docker)
11. [Acesso à API pelo celular](#11-acesso-à-api-pelo-celular)
12. [Troubleshooting](#12-troubleshooting)

Também: [Backend fora do Docker](#backend-fora-do-docker), [Flyway e Hibernate](#flyway-e-hibernate) e [Segurança](#segurança-e-credenciais).

## 1. Pré-requisitos

- Docker Desktop (Windows) ou Docker Engine com o plugin Compose v2.
- **Docker Compose 2.17.0 ou superior**, necessário para `additional_contexts` no build do frontend. BuildKit deve estar habilitado (é o padrão no Docker Desktop).
- Git.
- Node.js LTS, apenas para rodar o mobile (seção 10).
- JDK 21, apenas para rodar o backend fora do Docker.

Valide a versão e a configuração (em `infra/`):

```powershell
cd infra
docker compose version
docker compose config
docker compose build frontend
```

- `docker compose version` deve mostrar 2.17.0 ou superior.
- `docker compose config` deve terminar sem erro (exige o `.env` da seção 4) e mostrar `additional_contexts.infra` apontando para o caminho absoluto de `infra/`.
- `docker compose build frontend` deve concluir o `COPY --from=infra` sem erro.

## 2. Organização dos repositórios

`frontend/`, `backend/` e `mobile/` são clones Git independentes, cada um com `.git` próprio (sem submodules). O `.gitignore` da raiz ignora essas três pastas e os arquivos `.env`. Estrutura:

```
Easy-Eats-Docker/
├── .gitignore
├── README.md
├── frontend/   (clone independente)
├── backend/    (clone independente)
├── mobile/     (clone independente)
└── infra/
    ├── docker-compose.yml
    ├── Dockerfile.backend
    ├── Dockerfile.frontend
    ├── nginx.conf
    ├── .env            (não versionado)
    ├── .env.example
    └── docs/decisoes.md
```

Na raiz do repositório:

```powershell
git clone https://github.com/joao-vitor1007/Easy-Eats-Frontend frontend
git clone https://github.com/ZacML/Easy-Eats-Api-2 backend
git clone https://github.com/ZacML/Easy-Eats-app mobile
```

Para conferir que os segredos não serão versionados:

```powershell
git check-ignore -v infra/.env
git check-ignore -v infra/.env.example
```

O primeiro comando deve listar o arquivo como ignorado. O segundo não deve imprimir nada.

## 3. Docker Desktop e WSL2

No Windows:

1. Habilite o WSL2 (PowerShell como administrador): `wsl --install`. Reinicie o computador se solicitado.
2. Instale o [Docker Desktop](https://www.docker.com/products/docker-desktop/) e mantenha marcada a opção **Use WSL 2 based engine**.
3. Abra o Docker Desktop e aguarde o estado "Engine running".
4. Em um terminal novo, confira: `docker version` e `docker compose version`.

> Depois de instalar o Docker Desktop, o `docker.exe` pode não aparecer no PATH dos terminais já abertos. Feche e reabra o terminal (ou o VS Code) antes de executar `docker`.

## 4. Configuração do `.env`

Na raiz do repositório:

```powershell
Copy-Item infra/.env.example infra/.env
```

Edite `infra/.env`. Variáveis:

| Variável | Obrigatória | Descrição |
| --- | --- | --- |
| `POSTGRES_DB` | sim | Nome do banco. |
| `POSTGRES_USER` | sim | Usuário do Postgres (também usado pelo backend). |
| `POSTGRES_PASSWORD` | sim | Senha do Postgres (também usada pelo backend). |
| `JWT_SECRET` | sim | Segredo JWT em **Base64**. |
| `CORS_ALLOWED_ORIGINS` | sim | Origens permitidas, separadas por vírgula e sem espaços. |
| `FRONTEND_PORT` | não | Porta do frontend no host (padrão 4200). |
| `BACKEND_PORT` | não | Porta do backend no host (padrão 8080). |

`DB_URL`, `DB_USER` e `DB_PASSWORD` não ficam no `.env`: o Compose as deriva de `POSTGRES_*`. Se uma variável obrigatória faltar, o `docker compose` para com uma mensagem indicando qual é.

### Gerar o `JWT_SECRET` (Base64)

O backend espera o segredo em Base64. No PowerShell:

```powershell
$b = New-Object byte[] 48; [Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b); [Convert]::ToBase64String($b)
```

Cole o resultado em `JWT_SECRET`. Não reutilize segredos de exemplo nem o valor antigo do repositório.

### `CORS_ALLOWED_ORIGINS`

É obrigatória e não tem valor padrão no Compose. Para uso local: `http://localhost:4200`. Para acessar o frontend por outro dispositivo da rede, inclua a origem da LAN:

```
CORS_ALLOWED_ORIGINS=http://localhost:4200,http://192.168.0.10:4200
```

### Aviso: `POSTGRES_*` só valem na criação do volume

`POSTGRES_DB`, `POSTGRES_USER` e `POSTGRES_PASSWORD` só têm efeito quando o volume `db_data` é criado pela primeira vez. Alterá-las depois **não** muda as credenciais de um banco já inicializado, e o backend passará a falhar na autenticação. Veja [Persistência do banco](#9-persistência-do-banco) para trocar a senha.

## 5. Inicialização

```powershell
cd infra
docker compose up --build
```

Para rodar em segundo plano: `docker compose up --build -d`. Confira o estado:

```powershell
docker compose ps
curl.exe http://localhost:8080/actuator/health
```

`db` e `backend` devem estar `healthy` e o health deve responder `UP`. Endereços:

| Serviço | Endereço |
| --- | --- |
| Frontend | http://localhost:4200 |
| Backend (navegador do host) | http://localhost:8080 |
| Postgres (somente entre containers) | `db:5432` |

`db:5432` só existe dentro da rede Docker, usado pelo backend. A porta 5432 não é publicada no host. O navegador chama o backend em `http://localhost:8080`, porta publicada pelo backend. O Nginx não faz proxy para a API.

> No PowerShell, `curl` é alias de `Invoke-WebRequest`. Use `curl.exe`.

## 6. Encerramento

```powershell
cd infra
docker compose down
```

Sem `-v`, o volume do banco é preservado.

## 7. Logs

```powershell
cd infra
docker compose logs -f
docker compose logs -f backend
```

## 8. Reinício de serviços

```powershell
cd infra
docker compose restart backend
```

Substitua `backend` por `db` ou `frontend` conforme necessário. Mudanças em `infra/.env` exigem recriar o container (`docker compose up -d`), não apenas `restart`.

## 9. Persistência do banco

Os dados ficam no volume nomeado `db_data` (`easyeats_db_data` no host). `docker compose down` e `docker compose up` preservam os dados.

```powershell
docker volume ls
```

### Trocar a senha de um banco já criado

Alterar `POSTGRES_PASSWORD` no `.env` não basta. Altere a senha no banco e depois atualize o `.env`:

```powershell
cd infra
docker compose exec db psql -U <usuário> -d <banco>
```

No `psql`:

```sql
ALTER USER <usuário> WITH PASSWORD 'nova-senha';
```

Atualize `POSTGRES_PASSWORD` em `infra/.env` com a mesma senha e recrie o backend: `docker compose up -d backend`.

### Recriar o banco do zero

```powershell
cd infra
docker compose down -v
```

> **Aviso:** `down -v` apaga o volume `db_data` e causa **perda irreversível de todos os dados** do banco. Use só quando quiser recomeçar do zero. Na próxima subida, `POSTGRES_*` do `.env` valem de novo.

## 10. Mobile (Expo) fora do Docker

O app em `mobile/` não tem container nem serviço no Compose. Ele usa Expo, Development Builds e a variável `EXPO_PUBLIC_API_URL`. O guia completo está em `mobile/README.md` (development build, EAS, ADB).

```powershell
cd mobile
npm install
Copy-Item .env.example .env
```

Edite `mobile/.env` e defina `EXPO_PUBLIC_API_URL` (veja a seção 11). Depois:

```powershell
npx expo start --dev-client
```

Para cabo USB, faça também o túnel do Metro: `adb reverse tcp:8081 tcp:8081`.

As variáveis `EXPO_PUBLIC_*` são lidas quando o Metro inicia. Após alterar o `.env`, reinicie com cache limpo:

```powershell
npx expo start --dev-client -c
```

Pré-requisito: o backend deve estar em execução (seção 5). Gere um development build no celular conforme `mobile/README.md`.

## 11. Acesso à API pelo celular

| Cenário | `EXPO_PUBLIC_API_URL` |
| --- | --- |
| Celular físico no Wi-Fi | `http://<LAN_IP>:8080` (ex.: `http://192.168.x.x:8080`) |
| Emulador Android | `http://10.0.2.2:8080` |
| Celular por cabo USB | `http://localhost:8080`, após `adb reverse tcp:8080 tcp:8080` |

Descubra o `LAN_IP` do PC com `ipconfig` (endereço IPv4 do adaptador Wi-Fi).

O hostname `backend` só é válido entre containers. Não use `http://backend:8080` no celular.

### Firewall do Windows (porta 8080)

Para o celular no Wi-Fi alcançar o backend, libere a porta 8080 para redes privadas (PowerShell como administrador):

```powershell
New-NetFirewallRule -DisplayName "EasyEats API 8080" -Direction Inbound -Protocol TCP -LocalPort 8080 -Action Allow -Profile Private
```

Para remover a regra depois: `Remove-NetFirewallRule -DisplayName "EasyEats API 8080"`.

## Backend fora do Docker

O `application.properties` não tem valores padrão para credenciais. Para rodar o backend direto na máquina (por exemplo, `./mvnw spring-boot:run` em `backend/`), defina no ambiente do processo **`DB_URL`, `DB_USER`, `DB_PASSWORD` e `JWT_SECRET`**. Sem `DB_PASSWORD` ou `JWT_SECRET`, a aplicação falha na inicialização. `CORS_ALLOWED_ORIGINS` é opcional fora do Docker (o padrão libera `localhost` nas portas 3000, 4200, 5173 e 8100).

```powershell
$env:DB_URL = "jdbc:postgresql://localhost:5432/<banco>"
$env:DB_USER = "<usuário>"
$env:DB_PASSWORD = "<senha>"
$env:JWT_SECRET = "<segredo em Base64>"
cd backend
./mvnw spring-boot:run
```

Como a porta 5432 não é publicada pelo Compose, use um Postgres local próprio ou publique a porta por sua conta.

## Flyway e Hibernate

Resumo (detalhes em [`infra/docs/decisoes.md`](infra/docs/decisoes.md)):

- O Flyway executa as migrations antes do Hibernate. As migrations V3 e V4 usam `CREATE TABLE IF NOT EXISTS`.
- `ddl-auto=update` permanece ativo. Há risco de divergência entre o schema das migrations e o gerado pelo Hibernate, e o `update` não remove nem altera colunas existentes.
- Em banco existente sem `flyway_schema_history`, `baseline-on-migrate=true` e `baseline-version=0` fazem o Flyway criar a linha de baseline e aplicar as migrations seguintes.
- Uma migration com falha interrompe a inicialização do backend.
- `out-of-order=false`: migrations com versão inferior à última aplicada são rejeitadas.

## Segurança e credenciais

- Nenhum segredo real é versionado. `infra/.env` está no `.gitignore`; `infra/.env.example` só tem placeholders.
- O `jwt.secret` fixo e a senha `root` antigos estão no **histórico Git** do repositório da API. Considere-os comprometidos: **rotacione** o segredo JWT (gere um novo, seção 4) e troque qualquer senha de banco que tenha sido a antiga `root`.
- `management.endpoints.web.exposure.include=*` permanece e expõe todos os endpoints do Actuator. Não publique o backend na internet sem restringir isso.

## 12. Troubleshooting

### Porta ocupada

Erro como `Bind for 0.0.0.0:8080 failed: port is already allocated`.

```powershell
netstat -ano | findstr :8080
Get-Process -Id <PID>
```

Encerre o processo ou mude `BACKEND_PORT` / `FRONTEND_PORT` em `infra/.env` e rode `docker compose up -d` em `infra/`. Se mudar `BACKEND_PORT`, ajuste também `EXPO_PUBLIC_API_URL`; o frontend continua chamando `http://localhost:8080`.

### Backend sem conexão com o banco

```powershell
cd infra
docker compose ps
docker compose logs db
docker compose logs backend
```

- `db` precisa estar `healthy`.
- `password authentication failed`: a senha do `.env` difere da do volume já criado. Veja [Trocar a senha](#trocar-a-senha-de-um-banco-já-criado) ou, se puder perder os dados, `docker compose down -v`.
- Dentro do Docker a URL é `jdbc:postgresql://db:5432/<banco>`; `localhost` não funciona entre containers.

### Erro de CORS

O navegador mostra `blocked by CORS policy`.

```powershell
cd infra
docker compose config | Select-String CORS
```

A origem exata usada no navegador (esquema, host e porta) deve estar em `CORS_ALLOWED_ORIGINS`, separada por vírgula e sem espaços. Se acessar por `http://192.168.x.x:4200`, inclua essa origem. Depois de editar o `.env`: `docker compose up -d backend`.

### Senha do Postgres que não muda

`POSTGRES_PASSWORD` só vale na criação do volume. Use `ALTER USER` (seção 9) ou, perdendo os dados, `docker compose down -v` seguido de `docker compose up --build`.

### Bloqueio de firewall

O celular não alcança `http://<LAN_IP>:8080`.

```powershell
curl.exe http://localhost:8080/actuator/health
curl.exe http://<LAN_IP>:8080/actuator/health
Get-NetFirewallRule -DisplayName "EasyEats API 8080"
```

Se o primeiro funciona e o segundo falha, crie a regra de firewall da seção 11. Confira também se o celular e o PC estão na mesma rede e se a rede Wi-Fi não isola dispositivos.

### `docker` não é reconhecido

Feche e reabra o terminal após instalar o Docker Desktop e confirme que o Docker Desktop está em execução.
