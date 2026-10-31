# Rclone Google Drive Upload via Docker Compose Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Configurar um ambiente Docker Compose reprodutível e seguro para autenticar e realizar o upload sob demanda de arquivos locais para o Google Drive via Rclone.

**Architecture:** A solução utiliza a imagem oficial `rclone/rclone` com dois serviços definidos no `docker-compose.yml`: um serviço interativo (`config`) que expõe a porta `53682` para autenticação OAuth via navegador e salva o arquivo `rclone.conf` em volume persistente, e um serviço (`upload`) que executa `rclone copy` montando a pasta de dados local e o arquivo de configuração como volumes de leitura.

**Tech Stack:** Docker, Docker Compose, Rclone (imagem oficial `rclone/rclone:latest`).

**Spec:** [docs/superpowers/specs/2026-10-31-rclone-gdrive-upload-design.md](file:///Users/danieljunior/workspace/rclone_drive/docs/superpowers/specs/2026-10-31-rclone-gdrive-upload-design.md)

## Global Constraints

- Sistema operacional host: macOS.
- Execução estritamente sob demanda (sem cron ou daemons permanentes).
- Uso do comando aditivo `rclone copy` (sem deleção remota).
- Proteção estrita contra versionamento de credenciais (`.env` e `config/rclone.conf` ignorados no Git).
- Autenticação OAuth interativa via navegador através do mapeamento de porta `53682:53682`.

## Review Focus

- Arquivo de configuração ou `.env` acidentalmente rastreados pelo Git: `.gitignore` deve cobrir explicitamente `.env`, `*.env` e `config/rclone.conf`.
- Pasta local inexistente ou não definida no `.env`: Docker Compose deve apontar variáveis padrão claras para evitar falha silenciosa de montagem.
- Conflito de porta `53682` durante o fluxo de autorização web: serviço de configuração deve rodar de forma isolada e efêmera (`--rm`).
- Permissões de arquivo geradas pelo container no host macOS: volumes e arquivos de configuração devem ser legíveis pelo usuário local.
- Execução acidental de upload destrutivo: o serviço compose deve invocar explicitamente `copy` e nunca `sync`.

---

### Task 1: Environment & Security Scaffolding

**Files:**
- Create: `.gitignore`
- Create: `.env.example`
- Create: `config/.gitkeep`

**Interfaces:**
- Produces: Variáveis de ambiente (`LOCAL_PATH`, `RCLONE_REMOTE_NAME`, `REMOTE_PATH`) e regras de proteção no Git para credenciais e tokens.

- [ ] **Step 1: Create `.gitignore` to protect sensitive files and tokens**

Create `.gitignore` containing:
```gitignore
.env
*.env
config/rclone.conf
```

- [ ] **Step 2: Create `.env.example` with documented defaults**

Create `.env.example` defining:
```dotenv
# Caminho da pasta local que sera enviada (caminho absoluto ou relativo ao docker-compose.yml)
LOCAL_PATH=./data

# Nome do remote configurado no rclone (ex: gdrive)
RCLONE_REMOTE_NAME=gdrive

# Caminho da pasta no Google Drive onde os arquivos serao salvos (deixe vazio para raiz)
REMOTE_PATH=backup
```

- [ ] **Step 3: Create `config/.gitkeep` directory**

Create directory `config/` with an empty `.gitkeep` so the configuration folder structure is preserved in git without tracking `rclone.conf`.

- [ ] **Step 4: Verify git status ignores sensitive files**

Run: `git check-ignore -v .env config/rclone.conf`
Expected: Output confirms both `.env` and `config/rclone.conf` are matched and ignored by `.gitignore`.

---

### Task 2: Docker Compose Configuration

**Files:**
- Create: `docker-compose.yml`

**Interfaces:**
- Consumes: `.env` (ou defaults do Compose) e `./config` para armazenamento do `rclone.conf`.
- Produces: Serviços Compose `config` (autenticação interativa com porta 53682) e `upload` (execução sob demanda de `rclone copy`).

- [ ] **Step 1: Write `docker-compose.yml` defining `config` and `upload` services**

Define:
- `services.config`:
  - `image: rclone/rclone:latest`
  - `container_name: rclone_config`
  - `stdin_open: true`
  - `tty: true`
  - `ports: ["53682:53682"]`
  - `volumes: ["./config:/config/rclone"]`
  - `command: ["config"]`
- `services.upload`:
  - `image: rclone/rclone:latest`
  - `container_name: rclone_upload`
  - `volumes:`
    - `./config:/config/rclone:ro`
    - `${LOCAL_PATH:-./data}:/data:ro`
  - `command:`
    - `copy`
    - `/data`
    - `${RCLONE_REMOTE_NAME:-gdrive}:${REMOTE_PATH:-backup}`
    - `--progress`
    - `--fast-list`
    - `--drive-chunk-size=32M`

- [ ] **Step 2: Validate compose file syntax and config resolution**

Run: `docker compose config`
Expected: Exibe a configuração resolvida sem erros de validação sintática do Docker Compose.

---

### Task 3: Usage Documentation and Operational Validation

**Files:**
- Create: `README.md`

**Interfaces:**
- Consumes: Arquivos gerados nas Tasks 1 e 2 (`docker-compose.yml`, `.env.example`).
- Produces: Guia operacional completo para autenticação OAuth no Google Drive e execução de uploads (incluindo teste com `--dry-run`).

- [ ] **Step 1: Write `README.md` with complete step-by-step instructions**

Include:
- Pré-requisitos (Docker / Docker Compose instalado).
- Passo 1: Copiar `.env.example` para `.env` e ajustar caminhos.
- Passo 2: Executar `docker compose run --rm config` para criar o remote `gdrive` (tipo `drive`, autenticação automática no browser via localhost:53682).
- Passo 3: Testar com simulação (`--dry-run`):
  `docker compose run --rm upload copy /data ${RCLONE_REMOTE_NAME}:${REMOTE_PATH} --dry-run`
- Passo 4: Executar o upload definitivo sob demanda:
  `docker compose run --rm upload`

- [ ] **Step 2: Verify `docker compose run --rm upload --help` works cleanly**

Run: `docker compose run --rm upload --help` (ou `docker compose config`)
Expected: Saída válida do help do Rclone ou validação limpa do compose.
