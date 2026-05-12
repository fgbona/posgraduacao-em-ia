# Prompt

## Role
Você é um engenheiro DevOps/SRE sênior especialista em containerização Docker e deploy em Kubernetes.

## Task
Tenho um sistema chamado Lift que está rodando em VM atualmente e deve ser migrado para Kubernetes. Na vm, o codigo atualmente rodando é uma API Python/Flask na porta 8080, dependências declaradas em requirements.txt, e duas variáveis de ambiente que precisam estar presentes no runtime, DATABASE_URL e API_KEY. Em produção o serviço sobe com gunicorn --bind 0.0.0.0:8080 --workers 4 app:app.

Estrutura do projeto:

lift/
├── app.py
├── requirements.txt
├── lib/
│   ├── auth.py
│   └── storage.py
└── tests/
    └── test_app.py


requirements.txt

Flask==3.0.0
gunicorn==21.2.0
requests==2.31.0
python-dotenv==1.0.0
psycopg2-binary==2.9.9

## Format
Crie um Dockerfile para rodar o projeto com as melhores práticas de infraestrutura e segurança.


# Modelo
Claude sonnet 4.6


# Output
```
# =============================================================================
# Stage 1: builder
# Instala dependências em ambiente isolado para não contaminar a imagem final
# =============================================================================
FROM python:3.12-slim AS builder

# Evita geração de .pyc e garante output sem buffer (logs em tempo real)
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /build

# Copia apenas o requirements primeiro para aproveitar o cache de layers
# Se o requirements.txt não mudar, essa layer não é rebuilda
COPY requirements.txt .

# Instala as dependências no diretório /install para copiar depois
RUN pip install --upgrade pip \
    && pip install --prefix=/install -r requirements.txt


# =============================================================================
# Stage 2: runtime
# Imagem final mínima — só o que é necessário para rodar
# =============================================================================
FROM python:3.12-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    # Aponta o Python para os pacotes instalados no builder
    PYTHONPATH=/app \
    PATH="/install/bin:$PATH"

# Instala apenas dependências de sistema necessárias para psycopg2-binary
# e remove o cache do apt na mesma layer para não inflar a imagem
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        libpq5 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Cria usuário e grupo sem privilégios
# -r = system account, -g = grupo, -s /sbin/nologin = sem shell interativo
RUN groupadd -r lift \
    && useradd -r -g lift -s /sbin/nologin -d /app lift

# Copia os pacotes Python instalados no builder
COPY --from=builder /install /usr/local

WORKDIR /app

# Copia o código-fonte com ownership já definido
# Não copia tests/ — desnecessário em produção
COPY --chown=lift:lift app.py .
COPY --chown=lift:lift lib/ ./lib/

# Troca para o usuário sem privilégios antes de qualquer CMD/ENTRYPOINT
USER lift

# Documenta a porta exposta (não publica — isso é responsabilidade do K8s Service)
EXPOSE 8080

# Healthcheck: verifica se a API está respondendo
# --interval: frequência da checagem
# --timeout: tempo máximo de resposta
# --start-period: tempo de grace no startup
# --retries: falhas consecutivas para marcar como unhealthy
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/health')" \
    || exit 1

# DATABASE_URL e API_KEY são injetadas em runtime pelo Kubernetes (Secret/ConfigMap)
# Não defina valores default aqui — falha explícita é melhor que segredo vazio

# Entrypoint com exec form (array JSON) para garantir que o processo receba
# sinais do kernel diretamente (SIGTERM para graceful shutdown no K8s)
CMD ["gunicorn", \
     "--bind", "0.0.0.0:8080", \
     "--workers", "4", \
     "--timeout", "60", \
     "--access-logfile", "-", \
     "--error-logfile", "-", \
     "--log-level", "info", \
     "app:app"]
```

Decisões de design
Multi-stage build
A separação em dois stages (builder / runtime) é a prática mais importante aqui:

O builder instala pip, compiladores e dependências de build — nada disso vai para produção
O runtime copia apenas os .whl já instalados, resultando numa imagem final significativamente menor e com menor superfície de ataque

Segurança
PráticaMotivoUsuário lift sem privilégiosContainment: processo não roda como root/sbin/nologin no useraddSem shell interativo — impossível docker exec -it ... /bin/bash acidentalmentetests/ não copiadoCódigo de teste não tem lugar em produçãoSem ENV DATABASE_URL / ENV API_KEYSegredos são injetados pelo K8s em runtime; valor hardcoded no Dockerfile vaza via docker inspect e docker historyapt-get clean && rm -rf /var/lib/apt/lists/*Reduz tamanho e elimina metadados de pacotes desnecessários

Sinais e graceful shutdown
O CMD usa exec form (array JSON) em vez de shell form (CMD "gunicorn ..."). Com shell form, o PID 1 seria /bin/sh, e o SIGTERM do Kubernetes no rolling update não chegaria ao Gunicorn — o pod seria encerrado forçosamente após o terminationGracePeriodSeconds. Com exec form, o Gunicorn é o PID 1 e trata o sinal corretamente.

Cache de layers otimizado
O COPY requirements.txt . vem antes do COPY app.py, porque o requirements muda com muito menos frequência que o código. O Docker reusa a layer do pip install a menos que o arquivo de dependências mude — builds iterativos ficam muito mais rápidos.

Healthcheck
O HEALTHCHECK instrui o Docker runtime (e o Kubernetes, se você não usar liveness probe no manifest) a verificar a saúde do container. Para o K8s, o recomendado é configurar livenessProbe e readinessProbe no Deployment — o healthcheck do Dockerfile é um fallback útil para ambientes sem orquestração.


# Justificativa

Role: Defini que a role deveria ser um desenvolvedor senior, especialista em Kubernetes e docker pela experiencia necessaria e complexidade do contexto.
Task: Dei um contexto mais simplorio, dando algumas informações úteis para que o modelo pudesse ser o mais acertivo possível em sua resposta, como onde eu estou e para onde quero ir.
Format: Arquivo docker seguindo as boas praticas de infraestrutura e segurança mais explicação sobre a decisão de design definido, contento multi-stage building, segurança, sinais e graceful shutdown, cache de layers otimizado e healthcheck.
