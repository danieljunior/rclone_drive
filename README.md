# Rclone Google Drive Upload via Docker

Configuração para autenticar e realizar uploads sob demanda de uma pasta local para o Google Drive utilizando Docker Compose e a imagem oficial do Rclone.

---

## 📋 Pré-requisitos

- **Docker Desktop** (com Docker Compose v2) instalado e rodando na sua máquina.
- Uma conta Google com acesso ao Google Drive.

---

## 🚀 Passo a Passo de Configuração

### 1. Configurar variáveis de ambiente

Copie o arquivo `.env.example` para `.env`:

```bash
cp .env.example .env
```

Edite o arquivo `.env` para ajustar os caminhos e nomes:

- `LOCAL_PATH`: Caminho da pasta local que deseja enviar (pode ser relativo como `./data` ou absoluto como `/Users/seu-usuario/Documentos/minha-pasta`).
- `RCLONE_REMOTE_NAME`: Nome que você dará à conexão com o Google Drive (padrão: `gdrive`).
- `REMOTE_PATH`: Nome da pasta de destino dentro do seu Google Drive (ex: `backup`).

---

### 2. Autenticar no Google Drive (apenas na 1ª vez)

Execute o comando interativo de configuração:

```bash
docker compose run --rm config
```

Siga os prompts do Rclone:

1. Digite `n` para criar um **New remote**.
2. **name**: Digite `gdrive` (ou o mesmo nome definido em `RCLONE_REMOTE_NAME` no seu `.env`).
3. **Storage**: Digite `drive` (para selecionar Google Drive).
4. **client_id** e **client_secret**: Pressione `Enter` para deixar em branco (ou informe os seus se possuir um projeto próprio no GCP).
5. **scope**: Digite `1` (Full access to all files).
6. **service_account_file**: Pressione `Enter` para deixar em branco.
7. **Edit advanced config**: Digite `n`.
8. **Use auto config**: Digite `y`. O container abrirá a porta `53682`. No terminal será exibido um link ou o navegador abrirá automaticamente para você fazer login na sua conta Google e conceder acesso.
9. **Configure this as a Shared Drive (Team Drive)**: Digite `n` (a menos que use um Drive compartilhado corporativo).
10. Confirme com `y` e depois digite `q` para sair do menu.

> **Nota de Segurança:** As credenciais serão salvas em `./config/rclone.conf`. Este arquivo e o `.env` já estão adicionados ao `.gitignore` para nunca serem versionados.

---

### 3. Testar com simulação (`--dry-run`)

Antes de realizar a transferência real, você pode simular o envio para verificar quais arquivos seriam copiados:

```bash
docker compose run --rm upload copy /data gdrive:backup --dry-run
```

---

### 4. Executar o Upload sob Demanda

Sempre que desejar realizar o upload dos arquivos da pasta configurada para o Google Drive, execute:

```bash
docker compose run --rm upload
```

O container irá:
1. Montar a pasta local configurada como somente leitura (`/data:ro`).
2. Copiar os arquivos novos e alterados para o Google Drive (`rclone copy`), sem deletar arquivos remotos.
3. Exibir a barra de progresso em tempo real (`--progress`).
4. Encerrar a execução automaticamente ao terminar a transferência.
