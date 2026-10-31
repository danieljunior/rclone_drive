# Design Doc: Rclone Google Drive Upload via Docker Compose

## 1. Intent and Why
O usuário deseja realizar o upload de arquivos de uma pasta local para o Google Drive sob demanda utilizando um container Docker com o Rclone, garantindo que o ambiente seja isolado, reprodutível e seguro sem a necessidade de instalar binários do Rclone diretamente no sistema operacional host.

## 2. Goals, Non-Goals and Anti-Goals
- **Goals:**
  - Configuração sob demanda: o container inicializa, executa a transferência e desliga.
  - Autenticação inicial simples: suporte para rodar o processo de autorização OAuth do Google Drive via container expondo a porta de callback (`53682`) para captura do token no navegador.
  - Transferência segura e aditiva: uso do comando `rclone copy` (preserva arquivos existentes na nuvem sem deletar nada).
  - Isolamento de credenciais: o arquivo de configuração gerado (`rclone.conf`) fica em um volume montado e ignorado pelo controle de versão (`.gitignore`).
- **Non-Goals:**
  - Sincronização contínua ou agendamento via cron dentro do container (o escopo é estritamente execução sob demanda).
  - Espelhamento bidirecional ou deleção remota (`sync`).
- **Anti-Goals:**
  - Expor credenciais ou tokens OAuth no histórico do Git ou em imagens Docker públicas.
  - Criar imagens Docker customizadas pesadas quando a imagem oficial `rclone/rclone` é suficiente.

## 3. Constraints
- Sistema operacional host: macOS.
- Dependência: Docker Desktop (com suporte a `docker compose`).
- Autenticação OAuth do Google Drive requer uma porta local acessível pelo navegador para interceptar o redirect de callback (porta padrão do Rclone: `53682`).

## 4. Approaches Considered
- **Abordagem 1: Docker Compose declarativo com `.env` (Escolhida)**
  - Imagem oficial `rclone/rclone`.
  - Serviços dedicados ou perfis no Compose:
    - Serviço para autenticação (`rclone config`) com porta `53682` mapeada.
    - Serviço para upload (`rclone copy`) com volumes de dados e configuração.
  - *Tradeoffs:* Exige `docker compose`, mas é limpo, declarativo, versionável e padronizado.
- **Abordagem 2: Script wrapper (Shell/Bash) em torno de `docker run`**
  - Scripts `upload.sh` e `config.sh` com comandos `docker run`.
  - *Tradeoffs:* Menos declarativo, depende de scripts de shell locais.
- **Abordagem 3: Dockerfile customizado**
  - Criação de imagem própria derivada de `rclone/rclone`.
  - *Tradeoffs:* Adiciona etapa de build desnecessária para uma ferramenta que já funciona perfeitamente via imagem oficial.

## 5. Architecture & Implementation Decisions
- **Estrutura de Diretórios:**
  - `docker-compose.yml`: Definição dos serviços `config` (autenticação interativa) e `upload` (execução da cópia).
  - `.env.example` e `.env`: Variáveis para configurar o caminho da pasta local (`LOCAL_PATH`), nome do remote no Rclone (`RCLONE_REMOTE_NAME=gdrive`) e pasta de destino no Drive (`REMOTE_PATH`).
  - `config/`: Diretório mapeado para `/config/rclone` onde ficará o `rclone.conf`.
  - `.gitignore`: Ignorará `config/rclone.conf` e `.env` para proteção de credenciais.
  - `README.md`: Instruções passo a passo de como rodar o comando de autenticação pela primeira vez e como executar os uploads posteriores.

## 6. What's Left to the Builder
- Definir flags adicionais recomendadas no comando de cópia (ex: `--progress`, `--fast-list`, `--drive-chunk-size=32M`).
- Ajustar permissões e mapeamento de usuário (UID/GID) para evitar problemas de permissão em arquivos criados pelo container no macOS.
