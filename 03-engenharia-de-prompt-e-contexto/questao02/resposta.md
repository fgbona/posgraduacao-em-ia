# Prompt

## Role
Você é um SRE sênior especialista em Cloud AWS.

## Task
Crie um script em Bash para fazer backup de uma database postgresql na AWS que roda em uma EC2. O script precisa de seguir uma ordem passo a passo:

- Fazer o dump com `pg_dump`
- Compactar com `gzip`
- Fazer upload do arquivo para o bucket S3 `hvt-ledger-backups` via CLI com `aws s3 cp`
- Depois de 30 dias no S3, remover os arquivos mais antigos
- Registrar as execuções do script com timestamp no arquivo `/var/log/ledger-backup.log`
- Definir `set -e` no início do script para, em caso de erro, parar a execução e mostrar a falha

Segue as variáveis do ambiente:

- **Host:** `ledger-db.internal.hvt.io`
- **Porta:** `5432`
- **Banco:** `ledger_prod`
- **Usuário de backup:** `backup_user`
- **Senha:** variável de ambiente `PGPASSWORD`, populada pelo AWS Secrets Manager via IAM role da instância
- **Região AWS:** `us-east-1`
- **SO da instância:** Ubuntu 22.04 LTS
- **Diretório de trabalho com 80 GB livres:** `/var/backups/ledger`
- **Tamanho médio atual do dump compactado:** ~12 GB

## Format
O script bash deve ser comentado em cada instrução feita para ter melhor qualidade para dar manutenção. Prefira o uso de functions, se for parte das melhores práticas.

---

# Modelo

**Claude Sonnet 4.6**

---

# Output

```bash
#!/usr/bin/env bash
# =============================================================================
# ledger_backup.sh
# Backup automatizado do banco PostgreSQL ledger_prod → S3
#
# Fluxo:
#   1. Valida dependências e variáveis obrigatórias
#   2. Executa pg_dump com compressão gzip em streaming
#   3. Faz upload do arquivo para s3://hvt-ledger-backups
#   4. Remove arquivos locais temporários
#   5. Remove objetos S3 com mais de 30 dias
#   6. Registra cada etapa em /var/log/ledger-backup.log
#
# Pré-requisitos na instância:
#   - postgresql-client (pg_dump)
#   - awscli v2
#   - IAM role com GetSecretValue + s3:PutObject + s3:ListBucket + s3:DeleteObject
#   - Variável PGPASSWORD exportada antes de chamar o script
#     (via aws secretsmanager get-secret-value, ou User Data / wrapper)
#
# Uso:
#   export PGPASSWORD=$(aws secretsmanager get-secret-value \
#       --secret-id ledger/backup_user --query SecretString --output text)
#   bash ledger_backup.sh
# =============================================================================

set -euo pipefail
# -e  → aborta em qualquer erro de comando
# -u  → trata variáveis não definidas como erro
# -o pipefail → propaga falha em qualquer segmento de pipe

# =============================================================================
# CONFIGURAÇÕES — altere apenas esta seção para adaptar a outros ambientes
# =============================================================================
readonly DB_HOST="ledger-db.internal.hvt.io"
readonly DB_PORT="5432"
readonly DB_NAME="ledger_prod"
readonly DB_USER="backup_user"
readonly AWS_REGION="us-east-1"
readonly S3_BUCKET="hvt-ledger-backups"
readonly S3_PREFIX="postgresql"                        # pasta lógica dentro do bucket
readonly BACKUP_DIR="/var/backups/ledger"              # diretório local de trabalho (80 GB livres)
readonly LOG_FILE="/var/log/ledger-backup.log"
readonly RETENTION_DAYS=30                             # dias de retenção no S3
readonly MIN_FREE_GB=20                                # mínimo de espaço livre exigido antes do dump
readonly TIMESTAMP=$(date +"%Y%m%d_%H%M%S")           # ex: 20250512_143000
readonly BACKUP_FILENAME="ledger_prod_${TIMESTAMP}.sql.gz"
readonly BACKUP_PATH="${BACKUP_DIR}/${BACKUP_FILENAME}"
readonly S3_KEY="${S3_PREFIX}/${BACKUP_FILENAME}"

# =============================================================================
# FUNÇÕES UTILITÁRIAS
# =============================================================================

# log_info MESSAGE
# Grava mensagem com timestamp no log e também no stdout
log_info() {
    local msg="[INFO]  $(date '+%Y-%m-%d %H:%M:%S') — $*"
    echo "$msg" | tee -a "${LOG_FILE}"
}

# log_error MESSAGE
# Grava erro com timestamp no log e no stderr
log_error() {
    local msg="[ERROR] $(date '+%Y-%m-%d %H:%M:%S') — $*"
    echo "$msg" | tee -a "${LOG_FILE}" >&2
}

# log_separator
# Linha divisória para facilitar leitura do log
log_separator() {
    echo "$(printf '=%.0s' {1..72})" | tee -a "${LOG_FILE}"
}

# =============================================================================
# FUNÇÃO: validate_environment
# Verifica dependências, variáveis e espaço em disco antes de iniciar o backup
# =============================================================================
validate_environment() {
    log_info "Validando ambiente..."

    # Verifica se pg_dump está disponível
    if ! command -v pg_dump &>/dev/null; then
        log_error "pg_dump não encontrado. Instale: sudo apt-get install -y postgresql-client"
        exit 1
    fi

    # Verifica se awscli está disponível
    if ! command -v aws &>/dev/null; then
        log_error "AWS CLI não encontrado. Instale conforme: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html"
        exit 1
    fi

    # Verifica se PGPASSWORD está definida (obrigatória para autenticação sem prompt)
    if [[ -z "${PGPASSWORD:-}" ]]; then
        log_error "Variável PGPASSWORD não definida. Exporte-a a partir do Secrets Manager antes de executar o script."
        exit 1
    fi

    # Cria o diretório de backup se não existir
    mkdir -p "${BACKUP_DIR}"

    # Verifica espaço em disco disponível no diretório de trabalho
    local free_gb
    free_gb=$(df -BG "${BACKUP_DIR}" | awk 'NR==2 {gsub("G",""); print \$4}')
    if (( free_gb < MIN_FREE_GB )); then
        log_error "Espaço insuficiente em ${BACKUP_DIR}: ${free_gb} GB livres (mínimo exigido: ${MIN_FREE_GB} GB)."
        exit 1
    fi

    log_info "Ambiente validado. Espaço disponível em ${BACKUP_DIR}: ${free_gb} GB."
}

# =============================================================================
# FUNÇÃO: run_dump
# Executa pg_dump com saída diretamente em gzip (streaming — sem arquivo SQL intermediário)
# ~12 GB compactado estimado; streaming evita uso duplo de disco
# =============================================================================
run_dump() {
    log_info "Iniciando dump do banco '${DB_NAME}' em ${DB_HOST}:${DB_PORT}..."

    # pg_dump flags relevantes:
    #   --format=custom  → formato binário; mais eficiente que plain text
    #   --no-password    → força leitura de PGPASSWORD (sem prompt interativo)
    #   --verbose        → registra progresso no stderr (capturado pelo log)
    # O output é redirecionado para gzip que salva diretamente no BACKUP_PATH
    pg_dump \
        --host="${DB_HOST}" \
        --port="${DB_PORT}" \
        --username="${DB_USER}" \
        --dbname="${DB_NAME}" \
        --format=custom \
        --no-password \
        --verbose \
        2>>"${LOG_FILE}" \
    | gzip --fast > "${BACKUP_PATH}"
    # --fast (nível 1) prioriza velocidade; para maior compressão use gzip -9 (mais lento)

    # Verifica se o arquivo foi criado e tem tamanho não-zero
    if [[ ! -s "${BACKUP_PATH}" ]]; then
        log_error "Arquivo de backup vazio ou não criado: ${BACKUP_PATH}"
        exit 1
    fi

    local file_size
    file_size=$(du -sh "${BACKUP_PATH}" | cut -f1)
    log_info "Dump concluído. Arquivo: ${BACKUP_PATH} (${file_size})."
}

# =============================================================================
# FUNÇÃO: upload_to_s3
# Faz upload do arquivo de backup para o S3 usando aws s3 cp
# Usa --storage-class STANDARD_IA para backups (custo mais baixo para arquivos raramente lidos)
# =============================================================================
upload_to_s3() {
    log_info "Iniciando upload para s3://${S3_BUCKET}/${S3_KEY}..."

    aws s3 cp "${BACKUP_PATH}" "s3://${S3_BUCKET}/${S3_KEY}" \
        --region "${AWS_REGION}" \
        --storage-class STANDARD_IA \
        --no-progress \
        2>>"${LOG_FILE}"
    # --no-progress suprime a barra de progresso no log (mantém saída limpa)

    log_info "Upload concluído: s3://${S3_BUCKET}/${S3_KEY}"
}

# =============================================================================
# FUNÇÃO: cleanup_local
# Remove o arquivo de backup local após upload bem-sucedido para liberar espaço
# =============================================================================
cleanup_local() {
    log_info "Removendo arquivo local temporário: ${BACKUP_PATH}"
    rm -f "${BACKUP_PATH}"
    log_info "Arquivo local removido."
}

# =============================================================================
# FUNÇÃO: rotate_s3_backups
# Lista todos os objetos no prefixo S3 e exclui os que têm mais de RETENTION_DAYS dias
# Usa aws s3api list-objects-v2 para obter LastModified com precisão
# =============================================================================
rotate_s3_backups() {
    log_info "Verificando objetos S3 mais antigos que ${RETENTION_DAYS} dias em s3://${S3_BUCKET}/${S3_PREFIX}/..."

    # Data de corte no formato ISO 8601 (UTC)
    local cutoff_date
    cutoff_date=$(date -u -d "-${RETENTION_DAYS} days" +"%Y-%m-%dT%H:%M:%SZ")
    log_info "Data de corte: ${cutoff_date}"

    # Lista objetos do prefixo e filtra os mais antigos pela data LastModified
    local old_objects
    old_objects=$(aws s3api list-objects-v2 \
        --bucket "${S3_BUCKET}" \
        --prefix "${S3_PREFIX}/" \
        --region "${AWS_REGION}" \
        --query "Contents[?LastModified<='${cutoff_date}'].Key" \
        --output text 2>>"${LOG_FILE}")

    # Se não há objetos antigos, encerra a função
    if [[ -z "${old_objects}" || "${old_objects}" == "None" ]]; then
        log_info "Nenhum objeto antigo encontrado para remoção."
        return 0
    fi

    # Itera sobre cada objeto e faz a exclusão individualmente
    local deleted_count=0
    while IFS= read -r key; do
        [[ -z "${key}" ]] && continue  # ignora linhas vazias

        log_info "Removendo objeto antigo: s3://${S3_BUCKET}/${key}"
        aws s3 rm "s3://${S3_BUCKET}/${key}" \
            --region "${AWS_REGION}" \
            2>>"${LOG_FILE}"

        (( deleted_count++ )) || true
    done <<< "${old_objects}"

    log_info "Rotação concluída. Objetos removidos: ${deleted_count}."
}

# =============================================================================
# FUNÇÃO: handle_error
# Trap de erro — chamada automaticamente pelo set -e em caso de falha
# Registra linha e comando que causou o erro
# =============================================================================
handle_error() {
    local exit_code=$?
    local line_number=\$1
    log_error "Falha na linha ${line_number} (exit code: ${exit_code}). Verifique o log: ${LOG_FILE}"
    # Tenta remover arquivo local parcial para não ocupar espaço
    [[ -f "${BACKUP_PATH}" ]] && rm -f "${BACKUP_PATH}" && log_error "Arquivo local parcial removido: ${BACKUP_PATH}"
    exit "${exit_code}"
}

# Registra o trap apontando para handle_error com o número da linha atual
trap 'handle_error ${LINENO}' ERR

# =============================================================================
# MAIN — ponto de entrada do script
# =============================================================================
main() {
    log_separator
    log_info "=== INÍCIO DO BACKUP — ${TIMESTAMP} ==="
    log_info "Banco: ${DB_NAME} | Host: ${DB_HOST} | Bucket: s3://${S3_BUCKET}"

    validate_environment   # 1. Valida dependências, variáveis e espaço em disco
    run_dump               # 2. Executa pg_dump | gzip → arquivo local
    upload_to_s3           # 3. Sobe o arquivo para o S3
    cleanup_local          # 4. Remove o arquivo local temporário
    rotate_s3_backups      # 5. Exclui objetos S3 com mais de 30 dias

    log_info "=== BACKUP CONCLUÍDO COM SUCESSO — $(date '+%Y-%m-%d %H:%M:%S') ==="
    log_separator
}

# Chama main apenas se o script for executado diretamente (não via source)
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi
```

---

### Estrutura e Fluxo

O script é dividido em 6 funções com responsabilidade única, seguindo o princípio de separação de concerns — facilita manutenção e testes isolados.

| Função | Responsabilidade |
|---|---|
| `validate_environment` | Checa dependências, `PGPASSWORD`, espaço em disco |
| `run_dump` | `pg_dump` com pipe direto para `gzip` |
| `upload_to_s3` | `aws s3 cp` com storage class `STANDARD_IA` |
| `cleanup_local` | Remove o `.sql.gz` local após upload |
| `rotate_s3_backups` | Exclui objetos S3 com `LastModified` > 30 dias |
| `handle_error` | Trap de `ERR` — loga linha, limpa arquivo parcial |

---

### Decisões de SRE

- **`set -euo pipefail`** — mais robusto que apenas `-e`; o `pipefail` captura falhas dentro do `pg_dump | gzip`, que de outra forma seriam silenciadas.

- **Streaming sem arquivo intermediário** — o pipe `pg_dump | gzip > arquivo` evita escrever ~25-30 GB de SQL puro antes de compactar, crítico dado que o disco tem 80 GB livres e o dump compactado já é ~12 GB.

- **`pg_dump --format=custom`** — formato binário nativo do PostgreSQL, mais compacto e restaurável com `pg_restore` com seleção de objetos.

- **`STANDARD_IA` no S3** — adequado para backups: custo de armazenamento menor, com custo de recuperação aceitável para um restore pontual.

- **`s3api list-objects-v2` na rotação** — mais preciso que `aws s3 ls` para filtrar por data via query JMESPath.

---

### Agendamento via Cron *(exemplo diário às 01h00 UTC)*

```bash
# Wrapper que busca a senha no Secrets Manager antes de chamar o script
0 1 * * * root export PGPASSWORD=$(aws secretsmanager get-secret-value \
  --secret-id ledger/backup_user \
  --query SecretString \
  --output text \
  --region us-east-1) && bash /opt/scripts/ledger_backup.sh
```

---

# Justificativa

**Role:** Uma vez a role estabelecida, o modelo tem melhor direcionamento para a resposta do que for solicitado. Nesse caso, como temos EC2, optei por colocar um SRE especialista em Cloud AWS.

**Task:** Solicitei criar um script com detalhes que dessem ao modelo a possibilidade de ser o mais assertivo possível. Como a role foi bem definida, a task veio bem completa.

**Format:** Um script Bash comentado, utilizando melhores práticas, dando preferência por functions.
