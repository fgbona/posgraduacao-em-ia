# Prompt
## Context

Strickland é head de segurança e compliance, publicou o padrão interno de IaC que todo módulo Terraform novo precisa seguir.







Tags obrigatórias em todo recurso: Owner, CostCenter, Environment.



Prefixo hvt- nos nomes de recursos.



Todo bucket S3 com: encryption habilitada (SSE-S3 mínimo), versioning ativo, block public access total, logging configurado.



Variáveis de entrada em variables.tf com description e type obrigatórios.



## Action

Crie um novo módulo Terraform, reutilizável, para criar buckets S3 aderentes aos padrões informados. 



## Result

O módulo será consumido por todos os times da empresa, então precisa vir com um exemplo de uso. Todo o resultado deverá ser impresso em formato markdown explicativo.



## Example

Abaixo um exemplo de módulo existente na empresa que deve ser usado como referência de estilo e estrutura
```
variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string
}

locals {
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }
}

resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  tags = merge(local.common_tags, {
    Name = "hvt-vpc-${var.environment}"
  })
}
```


# Modelo
Gemini 3 Thinking

# Output
Com base nas diretrizes de segurança e compliance estabelecidas pelo Strickland, desenhei um módulo reutilizável que centraliza todas as regras de governança exigidas.

Para garantir a compatibilidade com as versões modernas do provedor AWS no Terraform (v4.x e v5.x), as configurações de criptografia, versionamento, bloqueio público e log foram separadas em recursos próprios acoplados ao bucket principal.

Abaixo está a estrutura completa do módulo, seguida de um exemplo prático de consumo.

**Estrutura do Módulo S3** (/modules/s3_compliant)
1. variables.tf
Conforme a regra, todas as variáveis possuem type e description explicitamente definidos.

Terraform
```
variable "bucket_base_name" {
  description = "Nome base do bucket S3. O módulo aplicará automaticamente o prefixo 'hvt-' e o sufixo do ambiente."
  type        = string
}

variable "owner" {
  description = "Equipe ou tag de identificação do dono/responsável pelo recurso (Mandatório)."
  type        = string
}

variable "cost_center" {
  description = "Centro de custo associado a este recurso para fins de faturamento (Mandatório)."
  type        = string
}

variable "environment" {
  description = "Ambiente de implantação do recurso (ex: dev, staging, prod) (Mandatório)."
  type        = string
}

variable "log_target_bucket" {
  description = "ID/Nome do bucket S3 centralizado onde os logs de acesso deste bucket serão armazenados."
  type        = string
}

variable "log_target_prefix" {
  description = "Prefixo opcional para organizar os caminhos dos logs de acesso dentro do bucket de auditoria."
  type        = string
  default     = "s3-access-logs/"
}
```
2. main.tf
Aqui garantimos a padronização do nome com o prefixo hvt-, a fusão das tags obrigatórias e as 4 camadas de segurança exigidas no S3.

Terraform
```
locals {
  # Garantia de tags obrigatórias exigidas pelo Compliance
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }

  # Padronização de nomenclatura interna: prefixo hvt- e sufixo de ambiente
  bucket_name = "hvt-${var.bucket_base_name}-${var.environment}"
}

# 1. Recurso Principal do Bucket
resource "aws_s3_bucket" "this" {
  bucket        = local.bucket_name
  tags          = local.common_tags
  force_destroy = false # Evita destruição acidental em produção
}

# 2. Ativação de Versionamento (Obrigatório)
resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration {
    status = "Enabled"
  }
}

# 3. Criptografia em repouso (SSE-S3 mínimo garantido com AES256)
resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# 4. Bloqueio Total de Acesso Público (Block Public Access Total)
resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# 5. Configuração de Logging de Acesso Ativo
resource "aws_s3_bucket_logging" "this" {
  bucket = aws_s3_bucket.this.id

  target_bucket = var.log_target_bucket
  target_prefix = "${var.log_target_prefix}${local.bucket_name}/"
}
```
3. outputs.tf
Facilita a exportação de atributos essenciais para outros módulos ou recursos da arquitetura.

Terraform
```
output "bucket_id" {
  description = "O nome/ID do bucket gerado pelo padrão interno."
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "O ARN (Amazon Resource Name) do bucket criado."
  value       = aws_s3_bucket.this.arn
}
```
**Exemplo de Uso (Como os times consumirão o módulo)**
Qualquer time que precisar de um bucket S3 agora chamará o bloco abaixo em sua stack (por exemplo, no arquivo main.tf da aplicação):

Terraform
```
# Exemplo de consumo do módulo interno de conformidade de S3
module "app_data_storage" {
  source = "./modules/s3_compliant" # Caminho local ou repositório Git do módulo

  # Nomenclatura Base
  bucket_base_name = "app-dados-usuarios"

  # Tags Obrigatórias de Governança (Strickland Policy)
  environment = "production"
  owner       = "Data-Engineering-Team"
  cost_center = "CC-9942-DATA"

  # Destino dos logs de auditoria (Pode apontar para um bucket central do time de Security)
  log_target_bucket = "hvt-security-logs-production"
  log_target_prefix = "s3-audit/"
}
```
**O que esse exemplo vai gerar na AWS?**
**Nome final do Bucket:** hvt-app-dados-usuarios-production

**Tags acopladas:** * Owner = Data-Engineering-Team

CostCenter = CC-9942-DATA

Environment = production

**Segurança aplicada:** Bucket 100% privado, com criptografia AES256 automática, histórico de versões ativo e logs sendo despejados no bucket central de segurança.

# Justificativa

**Context:** Estabeleci o padrão interno de IaC publicado por Strickland, listando os requisitos obrigatórios.

**Action:** Instrui o modelo a criar um módulo s3 reutilizável focando nos padrões.

**Result:** O módulo deve incluir um exemplo de uso, pois todos irão utilizar.

**Example:** Forneci a referência do módulo em que o modelo precisava se basear para construir o código.
