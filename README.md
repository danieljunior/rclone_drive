# Rclone Google Drive Upload via Docker

Configuração para autenticar e realizar uploads sob demanda de uma pasta local para o Google Drive utilizando Docker Compose e a imagem oficial do Rclone.

---

## 📋 Pré-requisitos

- **Docker Desktop** (com Docker Compose v2) instalado e rodando no macOS.
- Uma conta Google com acesso ao Google Drive.

---

## 🚀 Passo a Passo de Configuração

### 1. Configurar variáveis de ambiente

Copie o arquivo `.env.example` para `.env`:

```bash
cp .env.example .env
```

Edite o arquivo `.env` para ajustar os caminhos:

- `LOCAL_PATH`: Caminho da pasta local que deseja enviar (ex: `./data` ou caminho absoluto como `/Users/seu-usuario/Documentos/fotos`).
- `RCLONE_REMOTE_NAME`: Nome da conexão configurada no Rclone (padrão: `gdrive`).
- `REMOTE_PATH`: Nome da pasta de destino no Google Drive (ex: `backup`, ou deixe vazio para a raiz do Drive).

---

### 2. Autenticar no Google Drive (apenas na 1ª vez)

Execute o assistente interativo de configuração:

```bash
docker compose run --rm --service-ports config
```

> **Atenção:** A flag `--service-ports` garante que a porta `53682` fique acessível para o retorno automático do navegador.

Siga os prompts do Rclone:

1. Digite `n` para criar um **New remote**.
2. **name**: Digite `gdrive` (ou o mesmo nome definido em `RCLONE_REMOTE_NAME` no seu `.env`).
3. **Storage**: Digite `drive` (para Google Drive).
4. **client_id** e **client_secret**: Pressione `Enter` para deixar em branco (ou use suas próprias credenciais do GCP se possuir).
5. **scope**: Digite `1` (Full access to all files).
6. **service_account_file**: Pressione `Enter` para deixar em branco.
7. **Edit advanced config**: Digite `n`.
8. **Use auto config**:
   - **Opção A (Automática):** Digite `y`. O container abrirá a porta `53682`. O terminal fornecerá a URL ou abrirá o navegador para login e autorização na conta Google.
   - **Opção B (Modo Headless / Sem falhas):** Caso o navegador não consiga se comunicar com a porta do container, repita escolhendo `n`. O Rclone fornecerá um link do Google para você abrir no seu navegador, fazer login e colar o código de autorização gerado diretamente no terminal.
9. **Configure this as a Shared Drive (Team Drive)**: Digite `n` (a menos que use um drive compartilhado).
10. Confirme com `y` para salvar e `q` para sair.

> **Segurança:** O arquivo de credenciais é salvo em `./config/rclone.conf`. Esse diretório e o arquivo `.env` estão protegidos no `.gitignore`.

---

### 3. Testar com simulação (`--dry-run`)

Antes de realizar a transferência real, você pode simular o envio para verificar quais arquivos seriam copiados:

```bash
docker compose run --rm upload copy /data gdrive:backup --dry-run
```
*(Substitua `gdrive:backup` pelo nome do seu remote e pasta caso tenha alterado no `.env`)*

---

### 4. Executar o Upload sob Demanda

Sempre que desejar realizar o upload dos arquivos da pasta local para o Google Drive:

```bash
docker compose run --rm upload
```

O container irá:
1. Montar a pasta local como somente leitura (`/data:ro`).
2. Copiar arquivos novos e alterados para o Google Drive (`rclone copy`), sem deletar arquivos remotos.
3. Atualizar automaticamente tokens de acesso expirados na pasta `./config`.
4. Exibir o progresso e finalizar o container assim que a cópia for concluída.
