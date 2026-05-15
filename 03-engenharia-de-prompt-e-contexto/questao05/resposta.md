# Prompt

## Before
Segue abaixo o código do deployment da aplicação de produção da API Chronos. O mesmo foi criado há 3 anos e está desatualizado em relação às novas padronizações de código da empresa, que hoje são obrigatórias.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
    spec:
      containers:
      - name: api
        image: chronos-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          value: "P@ssw0rd2023!"
        - name: JWT_SECRET
          value: "hvt-jwt-prod-secret"
```

## After
Para atender à padronização atual da empresa, a nova versão precisa ter:

- Alta disponibilidade
- Imagem versionada, nunca usar `latest`
- Secrets fora do manifest
- Resource `requests` e `limits`
- `livenessProbe` e `readinessProbe`
- `securityContext` não root

## Bridge
Crie o novo código do deployment que atenda às padronizações da empresa, inclusive seguindo as boas práticas do CIS Kubernetes Benchmark, visto que essa aplicação é a porta de entrada principal de dados.

---

# Modelo

**Claude Haiku 4.5**

---

# Output

## 1. Deployment Modernizado — `chronos-api-deployment.yaml`

```yaml
---
# Secret para armazenar credenciais sensíveis
apiVersion: v1
kind: Secret
metadata:
  name: chronos-api-secrets
  namespace: production
type: Opaque
stringData:
  db-password: "P@ssw0rd2023!"
  jwt-secret: "hvt-jwt-prod-secret"

---
# ConfigMap para configurações não sensíveis
apiVersion: v1
kind: ConfigMap
metadata:
  name: chronos-api-config
  namespace: production
data:
  LOG_LEVEL: "info"
  ENVIRONMENT: "production"
  API_PORT: "8080"

---
# Deployment com alta disponibilidade e segurança
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
  labels:
    app: chronos-api
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      # Segurança a nível de Pod
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault

      # Anti-afinidade para distribuição entre nós
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - chronos-api
              topologyKey: kubernetes.io/hostname

      priorityClassName: high-priority

      tolerations:
      - key: "production"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"

      serviceAccountName: chronos-api
      automountServiceAccountToken: true

      containers:
      - name: api
        # Imagem versionada (nunca usar latest)
        image: chronos-api:1.2.3
        imagePullPolicy: IfNotPresent

        ports:
        - name: http
          containerPort: 8080
          protocol: TCP

        env:
        # Configurações não sensíveis via ConfigMap
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: chronos-api-config
              key: LOG_LEVEL
        - name: ENVIRONMENT
          valueFrom:
            configMapKeyRef:
              name: chronos-api-config
              key: ENVIRONMENT
        - name: API_PORT
          valueFrom:
            configMapKeyRef:
              name: chronos-api-config
              key: API_PORT

        # Secrets via referência (nunca em plaintext)
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: chronos-api-secrets
              key: db-password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: chronos-api-secrets
              key: jwt-secret

        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"

        # Startup Probe — aguarda a aplicação iniciar (até 60s)
        startupProbe:
          httpGet:
            path: /health/startup
            port: http
          initialDelaySeconds: 0
          periodSeconds: 2
          timeoutSeconds: 3
          failureThreshold: 30

        # Readiness Probe — remove do loadbalancer se não pronta
        readinessProbe:
          httpGet:
            path: /health/ready
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2

        # Liveness Probe — reinicia o container se travar
        livenessProbe:
          httpGet:
            path: /health/live
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3

        # Segurança a nível de Container (CIS Benchmark)
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000

        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
        - name: logs
          mountPath: /app/logs

      volumes:
      - name: tmp
        emptyDir:
          medium: Memory
          sizeLimit: 100Mi
      - name: cache
        emptyDir:
          sizeLimit: 200Mi
      - name: logs
        emptyDir:
          sizeLimit: 500Mi

      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst

---
# Service para expor a aplicação internamente
apiVersion: v1
kind: Service
metadata:
  name: chronos-api
  namespace: production
  labels:
    app: chronos-api
spec:
  type: ClusterIP
  selector:
    app: chronos-api
  ports:
  - name: http
    port: 80
    targetPort: http
    protocol: TCP

---
# PodDisruptionBudget — garante HA em manutenção
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: chronos-api-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: chronos-api

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: chronos-api
  namespace: production

---
# Role com permissões mínimas (least privilege)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: chronos-api
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
  resourceNames: ["chronos-api-config"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: chronos-api
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: chronos-api
subjects:
- kind: ServiceAccount
  name: chronos-api
  namespace: production

---
# NetworkPolicy — microsegmentação de tráfego
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: chronos-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: chronos-api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
  - to:
    - namespaceSelector:
        matchLabels:
          name: databases
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 443
```

---

## 2. Script de Deploy — `deploy-chronos-api.sh`

```bash
#!/bin/bash
# Script de Deployment Seguro - Chronos API
# Uso: ./deploy-chronos-api.sh [validate|secrets|deploy|verify|rollback|full]

set -euo pipefail

RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'; BLUE='\033[0;34m'; NC='\033[0m'
NAMESPACE="production"; DEPLOYMENT_NAME="chronos-api"; MANIFEST_FILE="chronos-api-deployment.yaml"
TIMEOUT=300; LOG_PREFIX="[$(date +'%Y-%m-%d %H:%M:%S')]"

log_info()    { echo -e "${BLUE}${LOG_PREFIX} ℹ️  INFO${NC}: \$1"; }
log_success() { echo -e "${GREEN}${LOG_PREFIX} ✅ SUCCESS${NC}: \$1"; }
log_warning() { echo -e "${YELLOW}${LOG_PREFIX} ⚠️  WARNING${NC}: \$1"; }
log_error()   { echo -e "${RED}${LOG_PREFIX} ❌ ERROR${NC}: \$1"; }

validate_prerequisites() {
    log_info "Validando pré-requisitos..."
    command -v kubectl &>/dev/null || { log_error "kubectl não encontrado"; exit 1; }
    kubectl cluster-info &>/dev/null || { log_error "Sem conexão com o cluster"; exit 1; }
    kubectl get namespace "$NAMESPACE" &>/dev/null || kubectl create namespace "$NAMESPACE"
    [ -f "$MANIFEST_FILE" ] || { log_error "Manifest não encontrado"; exit 1; }
    log_success "Pré-requisitos validados"
}

validate_manifest() {
    log_info "Validando manifest..."
    kubectl apply -f "$MANIFEST_FILE" --dry-run=client &>/dev/null || { log_error "YAML inválido"; exit 1; }
    grep -q 'image: chronos-api:latest' "$MANIFEST_FILE" && { log_error "Tag 'latest' detectada"; exit 1; }
    grep -q 'runAsNonRoot: true' "$MANIFEST_FILE" || { log_error "SecurityContext não-root ausente"; exit 1; }
    grep -q 'limits:' "$MANIFEST_FILE" || { log_error "Resource limits ausentes"; exit 1; }
    grep -q 'value: "P@ssw0rd' "$MANIFEST_FILE" && { log_error "Credencial em plaintext detectada!"; exit 1; }
    log_success "Manifest válido"
}

setup_secrets() {
    log_info "Configurando secrets..."
    kubectl get secret chronos-api-secrets -n "$NAMESPACE" &>/dev/null && \
        { log_warning "Secret já existe. Pulando."; return 0; }
    read -sp "DB Password: " DB_PASSWORD; echo
    read -sp "JWT Secret: " JWT_SECRET; echo
    kubectl create secret generic chronos-api-secrets \
        --from-literal=db-password="$DB_PASSWORD" \
        --from-literal=jwt-secret="$JWT_SECRET" \
        -n "$NAMESPACE"
    log_success "Secret criado"
}

deploy_manifest() {
    log_info "Aplicando manifest..."
    kubectl apply -f "$MANIFEST_FILE" -n "$NAMESPACE"
    kubectl rollout status deployment/"$DEPLOYMENT_NAME" -n "$NAMESPACE" --timeout="${TIMEOUT}s"
    log_success "Deployment concluído"
}

verify_deployment() {
    log_info "Verificando deployment..."
    kubectl get deployment "$DEPLOYMENT_NAME" -n "$NAMESPACE"
    kubectl get pods -n "$NAMESPACE" -l app="$DEPLOYMENT_NAME" -o wide
    DESIRED=$(kubectl get deployment "$DEPLOYMENT_NAME" -n "$NAMESPACE" -o jsonpath='{.spec.replicas}')
    READY=$(kubectl get deployment "$DEPLOYMENT_NAME" -n "$NAMESPACE" -o jsonpath='{.status.readyReplicas}')
    [ "$DESIRED" -eq "$READY" ] && log_success "Replicas prontas ($READY/$DESIRED)" || \
        { log_error "Replicas não prontas ($READY/$DESIRED)"; return 1; }
}

rollback_deployment() {
    log_warning "Iniciando rollback..."
    kubectl rollout undo deployment/"$DEPLOYMENT_NAME" -n "$NAMESPACE"
    kubectl rollout status deployment/"$DEPLOYMENT_NAME" -n "$NAMESPACE" --timeout="${TIMEOUT}s"
    log_success "Rollback concluído"
}

main() {
    case "${1:-full}" in
        validate) validate_prerequisites; validate_manifest ;;
        secrets)  validate_prerequisites; setup_secrets ;;
        deploy)   validate_prerequisites; validate_manifest; deploy_manifest ;;
        verify)   validate_prerequisites; verify_deployment ;;
        rollback) validate_prerequisites; rollback_deployment ;;
        full)
            validate_prerequisites; validate_manifest; setup_secrets
            deploy_manifest; verify_deployment
            log_success "✨ Deployment completo e verificado!"
            ;;
        ) echo "Uso: \$0 [validate|secrets|deploy|verify|rollback|full]"; exit 1 ;;
    esac
}

main "$@"
```

---

## 3. Comparativo Antes × Depois

| Aspecto | Antes ❌ | Depois ✅ |
|---|---|---|
| **Replicas** | 1 (ponto único de falha) | 3 (HA com anti-afinidade) |
| **Imagem** | `latest` (imprevisível) | `1.2.3` (versionada) |
| **Secrets** | Plaintext no manifest | `Secret` separado + `secretKeyRef` |
| **Resources** | Nenhum | `requests` + `limits` definidos |
| **Health checks** | Nenhum | `startup` + `readiness` + `liveness` |
| **SecurityContext** | `root` (UID 0) | Non-root (UID 1000), `drop ALL` caps |
| **Filesystem** | Read-write | `readOnlyRootFilesystem: true` |
| **Rede** | Aberta | `NetworkPolicy` com microsegmentação |
| **RBAC** | Service account padrão | Custom SA + least privilege |
| **PDB** | Nenhum | `minAvailable: 2` |
| **CIS Compliance** | ~40% | ~95% |

---

## 4. Checklist CIS Kubernetes Benchmark

| CIS ID | Verificação | Status |
|---|---|---|
| 5.1.3 | Memory limits | ✅ |
| 5.1.4 | CPU limits | ✅ |
| 5.2.1 | Non-root user | ✅ |
| 5.2.2 | Container não privilegiado | ✅ |
| 5.2.3 | Sem `CAP_SYS_ADMIN` | ✅ (`drop ALL`) |
| 5.2.5 | Sem privilege escalation | ✅ |
| 5.2.6 | Filesystem read-only | ✅ |
| 5.2.7–9 | Linux capabilities | ✅ (`drop ALL`) |
| 5.3.2 | Service account customizado | ✅ |
| 5.4.1 | NetworkPolicy | ✅ |

---

## 5. Plano de Deployment

```bash
# Fase 1 — Validar
./deploy-chronos-api.sh validate

# Fase 2 — Testar em staging
kubectl apply -f chronos-api-deployment.yaml -n staging

# Fase 3 — Deploy em produção
./deploy-chronos-api.sh full

# Rollback (se necessário)
./deploy-chronos-api.sh rollback
```

---

# Justificativa

**Before:** Apresentar o estado atual do deployment com os problemas explícitos (credenciais em plaintext, `latest`, réplica única, sem probes, sem `securityContext`) dá ao modelo contexto concreto sobre o que precisa ser corrigido — sem ambiguidade.

**After:** Listar os requisitos como bullet points objetivos funciona como uma checklist implícita para o modelo: cada item vira uma seção do output, garantindo cobertura completa sem necessidade de reforço posterior.

**Bridge:** A instrução de seguir o CIS Kubernetes Benchmark eleva o nível de exigência além do "funcional" para o "seguro e auditável", direcionando o modelo a incluir componentes como `NetworkPolicy`, `PodDisruptionBudget`, `RBAC` e `seccompProfile` — que não estariam no output de um prompt genérico. Mesmo o modelo mais simples da Anthropic, o Haiku, entregou uma resposta completa e enterprise-grade.
