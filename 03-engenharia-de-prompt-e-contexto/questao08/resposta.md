# Prompt
# Context

Strickland é head de segurança e compliance, publicou o padrão interno de IaC que todo módulo Terraform novo precisa seguir.







Tags obrigatórias em todo recurso: Owner, CostCenter, Environment.



Prefixo hvt- nos nomes de recursos.



Todo bucket S3 com: encryption habilitada (SSE-S3 mínimo), versioning ativo, block public access total, logging configurado.



Variáveis de entrada em variables.tf com description e type obrigatórios.



# Action

Crie um novo módulo Terraform, reutilizável, para criar buckets S3 aderentes aos padrões informados. 



# Result

O módulo será consumido por todos os times da empresa, então precisa vir com um exemplo de uso. Todo o resultado deverá ser impresso em formato markdown explicativo.



# Example

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

# Justificativa
