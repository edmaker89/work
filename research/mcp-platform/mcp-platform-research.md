# Plataforma MCP (Model Context Protocol) — Pesquisa: Serviço Baseado em Freshservice

> **Objetivo**: Estruturar uma plataforma MCP multi-tenant para oferecer **MCP servers como serviço (B2B)** baseados no **Freshservice (ITSM)**, com billing, autenticação/SSO (OAuth 2.1 + PKCE), tokens, isolamento de dados, registry/catalog e provisionamento self-service. A plataforma **gerencia** o ciclo de vida completo: provisiona, autentica, monitora, cobra e governa os MCP servers Freshservice por tenant/ambiente.

---

## 1. Visão Geral & Oportunidade (Freshservice-Focused)

| Aspecto | Detalhamento |
|---------|-------------|
| **Produto Core** | **Freshservice MCP Server** — expõe tickets, assets, catálogo de serviços, change management, knowledge base, CMDB, workflows do Freshservice como *tools* MCP padronizadas. |
| **Arquitetura Core** | **MCP Host** (Claude Desktop, Cursor, agents custom) → **MCP Client** ↔ **Freshservice MCP Server** (nosso código, roda no nosso Data Plane). Transporte: **Streamable HTTP** (remoto, multi-tenant) + **STDIO** (dev local). |
| **Papéis da Nossa Plataforma** | 1. **Control Plane** (API, Provisioning, Billing, Registry, AuthZ, Admin Console) 2. **Data Plane** (MCP Gateway + Freshservice MCP Server containers por tenant/ambiente) 3. **Registry Privado** (versões, signing, security scan, tenant allowlist). |
| **Público-Alvo** | MSPs, consultorias ITSM, departamentos de TI enterprise, SaaS que querem expor Freshservice para agentes IA sem construir infra. |
| **Diferencial vs. Gateways Genéricos** | **Domain-specific**: tools nativas Freshservice (não wrappers genéricos), OAuth Freshservice nativo, rate limits alinhados à API Freshservice, schema de tools otimizado para ITSM, compliance LGPD/SOC2 herdado. |
| **Modelo de Negócio** | **Plataforma (Control Plane)** + **Freshservice MCP Servers gerenciados** por tenant/ambiente. Receita: subscription base + overage tool-calls + add-ons (connectors premium: Jira, Azure DevOps, Slack; SIEM streaming; custom domains; SLA enterprise; white-label). |

---

## 2. Arquitetura de Referência (Freshservice MCP Platform)

```mermaid
flowchart TB
    subgraph CP[Control Plane - SaaS Operator]
        API[API Gateway / Control API<br/>(REST + GraphQL)]
        PM[Provisioning Manager<br/>(Tenant lifecycle, FS MCP Server templates)]
        BM[Billing & Metering<br/>(Tool calls, MAU, add-ons)]
        RG[Registry & Catalog<br/>(Freshservice MCP Server versions)]
        AM[AuthZ Policy Engine<br/>(OPA/Cedar - TBAC/RBAC)]
        UI[Admin Console<br/>(React + Tailwind)]
        DB[(PostgreSQL<br/>Control Plane DB)]
        KV[(Redis<br/>Sessions, Cache, Rate Limit)]
    end

    subgraph DP[Data Plane - Tenant/Environment Isolation]
        direction TB
        subgraph T1[Tenant A - Shared Pool]
            GW1[MCP Gateway<br/>(Obot/MintMCP/Custom)]
            FS1[Freshservice MCP Server<br/>(Container pool compartilhado)]
            R1[(PostgreSQL + RLS<br/>Tenant A configs)]
            TK1[(Token Vault<br/>Freshservice OAuth tokens AES-256)]
        end
        subgraph T2[Tenant B - Dedicated]
            GW2[MCP Gateway Cluster<br/>HA]
            FS2[Freshservice MCP Server<br/>Dedicated replicas]
            R2[(PostgreSQL Dedicated<br/>+ Read Replicas)]
            TK2[(Token Vault Dedicated)]
            VPN[VPN / PrivateLink<br/>BYOC Option]
        end
        subgraph T3[Tenant C - BYOC]
            GW3[MCP Gateway on Customer VPC]
            FS3[Freshservice MCP Server on Customer VPC]
            R3[(Customer DB)]
            TK3[(Customer KMS)]
            TUN[Cloudflare Tunnel /<br/>Reverse Proxy]
        end
    end

    API --> PM
    PM --> GW1
    PM --> GW2
    PM --> TUN
    BM --> API
    RG --> API
    AM --> API
    UI --> API
    DB --> API
    KV --> API
```

### Componentes-chave (Freshservice-specific)

| Componente | Tecnologia Sugerida | Responsabilidade |
|------------|---------------------|------------------|
| **API Gateway** | Kong / Traefik | Rate limit (respeita limites Freshservice API), authZ, observabilidade, roteamento MCP |
| **Provisioning Manager** | Go + Temporal | Cria tenant/ambiente, provisiona gateway, deploya Freshservice MCP Server, configura OAuth Freshservice, emite `tenant.ready` |
| **Billing & Metering** | Stripe Billing + Metering / Lago | Tool-call tracking por tool Freshservice (ticket.create, asset.search, etc.), invoice, usage-based overage |
| **Policy Engine** | OPA (Rego) | TBAC/RBAC: qual tool Freshservice cada role pode chamar, rate limit por tool, data masking |
| **Admin Console** | React + Refine | Self-service: conectar domínio Freshservice, configurar OAuth, ver logs, rotacionar tokens, gerenciar catálogo tools |
| **MCP Gateway** | **Obot (open source)** / MintMCP / Custom | Termina Streamable HTTP, injeta auth Freshservice, roteia para servers, audit log, rate limit |
| **Freshservice MCP Server Runtime** | Container (K8s/Docker) - **nosso código Python/Go** | Implementa tools Freshservice: tickets.*, assets.*, catalog.*, changes.*, kb.*, cmdb.*, workflows.* |
| **Registry/Catalog** | PostgreSQL + pgvector | Versionamento do Freshservice MCP Server, security scan, signing, tenant allowlist |
| **Token Vault** | HashiCorp Vault / AWS Secrets Manager | Armazena **OAuth refresh tokens Freshservice** por tenant/ambiente, rotação automática |
| **Armazenamento Tenant** | PostgreSQL (RLS) ou DB dedicado | Configs Freshservice (domínio, API keys), mappings, audit logs |

---

## 3. Freshservice MCP Server — Tools & Capabilities

```mermaid
mindmap
  root((Freshservice<br/>MCP Server))
    Tickets
      ticket.create
      ticket.get
      ticket.list
      ticket.update
      ticket.add_note
      ticket.add_attachment
      ticket.approve
      ticket.associate_asset
    Assets
      asset.create
      asset.get
      asset.list
      asset.update
      asset.associate_ticket
      asset.lifecycle
    Catalog
      service_catalog.list
      service_item.get
      service_item.order
      approval.get
    Changes
      change.create
      change.get
      change.list
      change.approve
      change.task.*
    Knowledge Base
      article.search
      article.get
      article.create
      category.list
    CMDB
      ci.create
      ci.get
      ci.relationship.*
    Workflows
      workflow.list
      workflow.trigger
      workflow.get_execution
    Admin
      agent.list
      group.list
      requester.list
      business_hour.*
```

### Tool Definitions (MCP Spec)

**workflow.list**
```json
{
  "name": "workflow.list",
  "description": "Lista workflows disponíveis no tenant Freshservice",
  "inputSchema": {
    "type": "object",
    "properties": {
      "type": { "type": "string", "enum": ["automator", "engine", "catalog"] }
    }
  }
}
```

**workflow.trigger**
```json
{
  "name": "workflow.trigger",
  "description": "Dispara execução de workflow com parâmetros",
  "inputSchema": {
    "type": "object",
    "properties": {
      "workflow_id": { "type": "integer" },
      "input_parameters": { "type": "object" },
      "context": { "type": "object", "description": "ticket_id, requester_email, etc." }
    },
    "required": ["workflow_id"]
  }
}
```

**workflow.get_execution**
```json
{
  "name": "workflow.get_execution",
  "description": "Consulta status/resultado de execução",
  "inputSchema": {
    "type": "object",
    "properties": {
      "execution_id": { "type": "string" }
    },
    "required": ["execution_id"]
  }
}
```

> **Nota**: O *Workflow Engine* (visual, drag-drop) é **feature gated** — só aparece na API se o tenant tiver o add-on ativo (plano Enterprise/Forest). Verifique em **Admin → Workflow Automator → "New Workflow"**.

---

## 4. Modelo de Isolamento: **Um MCP Server por Ambiente (Sandbox/Prod)**

| Nível | O que é isolado | Como é feito (VPS → K8s) |
|-------|-----------------|---------------------------|
| **Organização (Cliente)** | Conta comercial, plano, billing, SSO config, owners | Row-level security no Postgres (`organization_id`) |
| **Ambiente** (Sandbox / Production) | Domínio Freshservice distinto, OAuth App distinto, quota de tool calls, feature flags, versão do MCP Server | **Schema Postgres dedicado** (`org_acme_sandbox`, `org_acme_prod`) + **Vault namespace/path** (`secret/org/acme/sandbox/fs-oauth`, `.../prod/`) + **Obot Skill registration** com config apontando para o path correto |
| **MCP Server Instance** | Processo/container que executa as tools | **Shared pool** (Starter/Professional) → mesmo container atende múltiplos ambientes via header `X-Environment-ID`; **Dedicated/BYOC** → container/pod exclusivo por ambiente |

**Por que por ambiente e não por cliente?**  
- Sandbox e Prod têm **domínios Freshservice diferentes**, **OAuth Apps diferentes**, **rate limits diferentes**, **dados diferentes**.  
- Permite promover versão do MCP Server de sandbox → prod com canary.  
- Isolamento de falha: bug no sandbox não derruba prod.  
- Billing e quota são configurados por ambiente.

---

## 5. Obot — Skill Registration Detalhado

O Obot trata **cada MCP Server** como uma **Skill**. No provisionamento de um ambiente, o Control Plane faz:

```bash
# 1. Registrar a skill (uma vez por versão do FS MCP Server)
curl -X POST http://obot:8080/api/v1/skills \
  -H "Authorization: Bearer $OBOT_ADMIN_TOKEN" \
  -d '{
    "name": "freshservice-mcp",
    "version": "1.2.0",
    "description": "Freshservice ITSM tools (tickets, assets, catalog, changes, kb, cmdb, workflows)",
    "transport": "streamable-http",
    "endpoint": "http://fs-mcp-server:8081/mcp",   # interno ao Docker/K8s
    "auth": {
      "type": "oauth2",
      "token_vault_path": "secret/org/{{org_id}}/{{env_id}}/fs-oauth"
    },
    "schema": { ... }   # JSON Schema completo das tools
  }'
```

```bash
# 2. Criar instância da skill para o ambiente (binding org+env → skill version)
curl -X POST http://obot:8080/api/v1/skill-instances \
  -H "Authorization: Bearer $OBOT_ADMIN_TOKEN" \
  -d '{
    "skill_name": "freshservice-mcp",
    "skill_version": "1.2.0",
    "tenant_id": "org_acme",
    "environment_id": "sandbox",
    "config": {
      "freshservice_domain": "acme-sandbox.freshservice.com",
      "vault_path": "secret/org/acme/sandbox/fs-oauth",
      "rate_limit_rpm": 100
    }
  }'
```

**Resultado**: Obot expõe endpoint único por ambiente  
`https://acme-sandbox.mcp.seudominio.com/mcp` (ou header `X-Tenant-ID: org_acme`, `X-Environment-ID: sandbox`) que roteia para a skill instance correta, injeta o token Freshservice do Vault e executa a tool.

**Vantagens dessa abordagem**:
- **Versionamento**: múltiplas versões do FS MCP Server convivem; promoção sandbox→prod = trocar `skill_version` na instância.
- **Hot reload**: novo deploy da skill version → `POST /api/v1/skill-instances/{id}/reload` sem reiniciar gateway.
- **Observabilidade por instância**: logs, métricas, traces segregados por `tenant_id+environment_id`.

---

## 6. Validação Inicial em VPS (Sem Kubernetes)

### Stack Mínima (Docker Compose)

```yaml
# docker-compose.yml — VPS single node
version: "3.8"
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: control_plane
      POSTGRES_USER: cp_user
      POSTGRES_PASSWORD: ${PG_PASSWORD}
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck: ["CMD-SHELL", "pg_isready -U cp_user"]

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes: [redisdata:/data]
    healthcheck: ["CMD", "redis-cli", "ping"]

  vault:
    image: hashicorp/vault:1.15
    cap_add: [IPC_LOCK]
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: ${VAULT_ROOT_TOKEN}
      VAULT_DEV_LISTEN_ADDRESS: "0.0.0.0:8200"
    ports: ["8200:8200"]
    command: server -dev

  obot:
    image: ghcr.io/obot-platform/obot:latest
    depends_on: [postgres, redis, vault]
    environment:
      DATABASE_URL: postgres://cp_user:***@postgres:5432/control_plane
      REDIS_URL: redis://redis:6379
      VAULT_ADDR: http://vault:8200
      VAULT_TOKEN: ${VAULT_ROOT_TOKEN}
      OBOT_AUTH_PROVIDERS: "oidc,github,google"
      OBOT_HOST: "0.0.0.0"
      OBOT_PORT: "8080"
    ports: ["8080:8080"]

  fs-mcp-server:
    build: ./fs-mcp-server
    depends_on: [vault]
    environment:
      VAULT_ADDR: http://vault:8200
      VAULT_TOKEN: ${VAULT_ROOT_TOKEN}
      LOG_LEVEL: info

  nginx:
    image: nginx:alpine
    ports: ["80:80", "443:443"]
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certbot:/etc/letsencrypt:ro
    depends_on: [obot]

  control-plane-api:
    build: ./control-plane-api
    depends_on: [postgres, redis, vault, obot]
    environment:
      DATABASE_URL: postgres://cp_user:***@postgres:5432/control_plane
      REDIS_URL: redis://redis:6379
      VAULT_ADDR: http://vault:8200
      VAULT_TOKEN: ${VAULT_ROOT_TOKEN}
      OBOT_URL: http://obot:8080
      STRIPE_SECRET_KEY: ${STRIPE_SECRET_KEY}
    ports: ["3000:3000"]

  admin-console:
    build: ./admin-console
    depends_on: [control-plane-api]
    ports: ["3001:3000"]

volumes:
  pgdata:
  redisdata:
```

### O que **perde** na VPS (recupera no K8s depois)

| Feature | VPS (Docker Compose) | K8s (Produção) |
|---------|----------------------|----------------|
| Isolamento forte rede/CPU/RAM por ambiente | ❌ Compartilhado | ✅ NetworkPolicy, ResourceQuota, LimitRange |
| HA automático (restart, node failure) | ❌ Single point | ✅ ReplicaSets, PodDisruptionBudget, multi-AZ |
| Secrets rotation automática | Manual / cron | ✅ Vault Agent Injector + CSI |
| TLS automático | Certbot manual | ✅ cert-manager + Let's Encrypt |
| Observabilidade unificada | Stdout + Loki local | ✅ Grafana Stack (Loki/Tempo/Mimir) + ServiceMonitor |
| Provisioning "< 60s" | Script shell + API | ✅ Temporal workflow + K8s Operator |

> **Nenhum bloqueador** — Control Plane API continua **igual**; só o *Provisioning Engine* troca "criar namespace K8s" por "criar schema PG + path Vault + entry nginx".

### Migração VPS → K8s (quando chegar a hora)

1. Mesmo `Dockerfile` do FS MCP Server → imagem no registry.  
2. `helm repo add obot https://charts.obot.ai` substitui container `obot`.  
3. Vault → modo HA (Raft) ou managed (HCP Vault / AWS Secrets Manager).  
4. Postgres → Cloud SQL / RDS / CloudNativePG operator.  
4. Provisioning Engine troca `docker run` / `psql` por `kubectl apply` (Namespace, Secret, Deployment, Service, NetworkPolicy).  
5. DNS wildcard (`*.mcp.seudominio.com`) → Ingress NGINX + cert-manager.

---

## 7. Stacks de Tecnologia & Linguagens Sugeridas

| Camada | Linguagem/Framework | Justificativa |
|--------|---------------------|---------------|
| **Control Plane API** | **Go (Chi/Gin) + Temporal** ou **TypeScript (NestJS/Fastify) + Temporal** | Go: performance, binário único, ecossistema K8s. TS: shared types com Admin Console, produtividade full-stack. |
| **Admin Console** | **React 18 + TypeScript + Refine (admin framework) + Tailwind + TanStack Query** | Refine acelera CRUD, auth, RBAC, audit logs; Tailwind para UI consistente. |
| **Provisioning Engine** | **Go + Temporal SDK** (workflows duráveis, retries, visibility) | Temporal garante idempotência, saga pattern, replay. |
| **Freshservice MCP Server** | **Python (FastMCP / mcp-sdk-python + FastAPI)** ou **Go (mcp-go + chi)** | Python: ecossistema Freshservice (requests/httpx), tipagem, dev rápido. Go: throughput, single binary, menor memória. |
| **Obot Gateway** | **Go (já vem compilado)** — configuramos via API/CRDs | Não precisamos mexer no código do Obot a menos que precisemos de plugin custom. |
| **Registry & Catalog** | **PostgreSQL + pgvector** (metadata + embeddings para busca semântica) | pgvector nativo, sem serviço extra. |
| **Token Vault** | **HashiCorp Vault** (dev mode na VPS, HA/Raft em prod) | Namespaces, KV v2, lease/rotation, audit log nativo. |
| **Billing & Metering** | **Stripe Billing + Stripe Metering** (ou **Lago** open-source se quiser self-host) | Stripe: portal self-service, invoices, tax, x402 crypto opcional. |
| **Observabilidade** | **OpenTelemetry Collector → Loki/Tempo/Mimir (Grafana Stack)** | Padrão CNCF, vendor-neutral. |
| **CI/CD** | **GitHub Actions + ArgoCD (GitOps)** | Arroba de ambientes, promoção sandbox→prod via PR. |
| **Infra as Code** | **Terraform (cloud infra) + Helm (K8s apps)** | Declarativo, versionado. |

### Decisão Go vs TypeScript no Control Plane

| Critério | Go | TypeScript |
|----------|----|------------|
| Performance/Throughput | ★★★★★ | ★★★☆☆ |
| Shared types com Frontend | ★★☆☆☆ (protobuf/OpenAPI) | ★★★★★ (compartilha Zod/TS types) |
| Ecossistema Temporal | ★★★★★ (SDK maduro) | ★★★★☆ (SDK bom) |
| Time-to-market (solo dev) | ★★★★☆ | ★★★★★ |
| Hiring (Brasil) | ★★★★☆ | ★★★★★ |
| **Recomendação** | Se prioriza **performance, custos de infra, single binary** | Se prioriza **velocidade de feature, shared types, full-stack único** |

> **Minha sugestão**: **TypeScript (NestJS/Fastify) + Temporal** para o Control Plane API + Admin Console no mesmo monorepo (shared packages `types`, `config`, `api-client`). O FS MCP Server em **Python (FastMCP)** — ecossistema Freshservice mais maduro, tipagem Pydantic, fácil para data engineers.

---

## 8. Monorepo vs Multi-repo — Estratégia para Múltiplos MCP Servers

### Cenário: Plataforma começa com Freshservice, depois adiciona **Jira MCP Server**, **PostgreSQL MCP Server**, **n8n MCP Server**, etc.

| Abordagem | Estrutura | Prós | Contras | Quando usar |
|-----------|-----------|------|---------|-------------|
| **Monorepo único** (`freshservice-mcp-platform/`) | `apps/control-plane-api`, `apps/admin-console`, `apps/fs-mcp-server`, `apps/jira-mcp-server`, `packages/shared-types`, `packages/tsconfig`, `infra/` | • Shared types & config (zero drift) <br>• Atomic commits cross-cutting (ex: mudança no schema de provisioning) <br>• CI/CD único, cache turbo <br>• Fácil refatoração de libs internas | • Build time cresce <br>• Time grande → conflitos de merge <br>• Deploy acoplado (precisa tooling: Turborepo, Nx, Bazel) | **Time ≤ 8 devs**, **ciclo de release sincronizado**, **forte acoplamento de domínio** (todos MCP servers seguem mesmo contrato de provisioning, auth, billing). |
| **Multi-repo (polyrepo)** | `control-plane/`, `admin-console/`, `fs-mcp-server/`, `jira-mcp-server/`, `shared-packages/` (publicado no npm/GitHub Packages) | • Isolamento de ciclo de vida <br>• Teams autônomos <br>• Deploy independente <br>• Menor blast radius | • Versionamento de shared packages (dependency hell) <br>• Duplicação de config (eslint, tsconfig, dockerfiles) <br>• Mudança cross-repo = PRs coordenados | **Time > 8 devs**, **domínios distintos** (ex: time ITSM cuida FS, time DevTools cuida Jira/GitHub), **releases independentes**. |
| **Híbrido (recomendado para vocês)** | **Monorepo para Control Plane + Admin Console + Provisioning + Infra** (core da plataforma) + **Repositórios separados por MCP Server** (`fs-mcp-server`, `jira-mcp-server`, `pg-mcp-server`) publicados como **Docker images + OpenAPI/JSON Schema** no Registry privado. | • Core da plataforma (billing, auth, provisioning) versionado junto <br>• Cada MCP Server evolui no seu ritmo, linguagens diferentes (Python, Go, TS) <br>• Control Plane consome apenas **artefatos imutáveis** (image + schema) → desacoplamento real <br>• Shared packages apenas para `types` (Zod/JSON Schema) e `api-client` TS/Go | • Dois fluxos de CI/CD (core + servers) <br>• Precisa de Registry privado robusto (já planejado) | **Vocês agora**: time pequeno, core acoplado, servers independentes. |

### Estrutura Híbrida Prática

```
# Repo 1: mcp-control-plane (monorepo)
mcp-control-plane/
├── apps/
│   ├── control-plane-api/      # Go ou TS + Temporal
│   ├── admin-console/          # React + Refine
│   └── provisioning-engine/    # Temporal workflows
├── packages/
│   ├── shared-types/           # Zod schemas, OpenAPI types
│   ├── api-client/             # TS/Go client p/ Control Plane API
│   └── config/                 # Env validation (zod)
├── infra/
│   ├── docker-compose.yml      # VPS dev
│   ├── helm/                   # K8s prod
│   └── terraform/              # Cloud infra
└── turbo.json / nx.json        # Turborepo ou Nx

# Repo 2: fs-mcp-server (independente)
fs-mcp-server/
├── src/                        # Python (FastMCP) ou Go
├── tests/
├── Dockerfile
├── pyproject.toml / go.mod
├── .github/workflows/ci.yml    # Build → push image → push schema to registry
└── mcp-manifest.json           # {name, version, tools[], auth, transport}

# Repo 3: jira-mcp-server (futuro)
jira-mcp-server/
└── ...

# Registry privado (Control Plane) armazena:
#   - Docker image digest (sha256)
#   - mcp-manifest.json (tools, inputSchemas, auth requirements)
#   - SBOM, scan results, cosign signature
```

**Fluxo de Release MCP Server** (ex: FS v1.3.0):
1. Dev faz PR no `fs-mcp-server` → CI roda testes, build image `ghcr.io/you/fs-mcp-server:1.3.0`, gera `mcp-manifest.json`, assina com cosign.
2. CI faz `POST /api/v1/registry/servers` no Control Plane (auth machine-to-machine) → Registry armazena versão, marca `status: staging`.
3. No Admin Console, operador clica **Promover para Produção** → Control Plane atualiza `skill_version` nas instâncias Obot dos ambientes desejados (canary → full).
4. Rollback = trocar `skill_version` anterior.

---

## 9. Padrões de Multi-Tenancy para Freshservice MCP (Atualizado)

```mermaid
graph LR
    A[Estratégia Multi-Tenant] --> B[Shared Gateway + Pool<br/>RLS PG + Vault Paths]
    A --> C[Gateway por Tenant<br/>(Namespace K8s / Docker Network)]
    A --> D[Gateway Cluster Dedicado<br/>(HA, Recursos Garantidos)]
    A --> E[BYOC<br/>Gateway no VPC Cliente]

    B --> B1[Baixo custo<br/>Alta densidade<br/>Rate limit compartilhado FS API]
    C --> C1[Isolamento forte<br/>Rate limit dedicado<br/>Custo médio]
    D --> D1[Isolamento total<br/>SLA + throughput garantido<br/>Custo alto]
    E --> E1[Data residency total<br/>Tokens FS nunca saem do VPC cliente<br/>Suporte complexo]
```

### Recomendação por Tier (Freshservice)

| Tier | Estratégia Gateway | Freshservice OAuth | Rate Limit FS API | SLA | Preço Base (BRL) |
|------|-------------------|-------------------|-------------------|-----|------------------|
| **Starter** | Shared Pool + RLS | Shared Vault + Paths | Pool compartilhado (100 req/min) | 99.5% | R$ 299/mês (até 10k tool calls) |
| **Professional** | Namespace K8s / Docker Network dedicado | Namespace Vault | Dedicado (500 req/min) | 99.9% | R$ 999/mês (até 100k tool calls) |
| **Enterprise** | Cluster Dedicado HA | Vault Dedicado + KMS | Dedicado (2000 req/min) | 99.95% | R$ 3.999/mês (até 1M tool calls) |
| **Dedicated/BYOC** | BYOC (Gateway no VPC cliente) | Customer KMS | Ilimitado (cliente controla) | Custom | Sob consulta |

> **Nota crítica**: Freshservice API tem rate limits por domínio. Tier Starter compartilha o limite; tiers superiores têm pools dedicados ou BYOC.

---

## 10. Modelo de Billing & Monetização (Freshservice MCP)

```mermaid
sequenceDiagram
    participant Customer
    participant Console
    participant API
    participant Stripe
    participant Metering
    Customer->>Console: Conecta domínio Freshservice / escolhe plano
    Console->>API: POST /organizations {name, plan, environments:[{name,type,domain,oauth}]}
    API->>Provisioning: Para cada env: provisiona gateway + FS MCP Server + OAuth config
    API->>Stripe: Create subscription + meter (tool_calls) por env
    Provisioning-->>API: env_id, gateway_url, client_credentials
    API-->>Console: Org pronta + credenciais MCP por ambiente
    loop Contínuo
        Metering->>Stripe: Report tool_calls por tool (ticket.create, workflow.trigger...)
        Stripe->>Customer: Invoice (base + overage por tool/ambiente)
    end
```

### Tabela de Preços (Exemplo BRL - Freshservice)

| Componente | Starter | Professional | Enterprise |
|------------|---------|--------------|------------|
| **Base/mês** | R$ 299 | R$ 999 | R$ 3.999 |
| **Tool calls incluídos** | 10.000 | 100.000 | 1.000.000 |
| **Tool call extra** | R$ 0,020/call | R$ 0,012/call | R$ 0,007/call |
| **Ambientes (sandbox+prod)** | 2 | 5 | 10 |
| **Concurrent connections** | 5 | 25 | 100 |
| **Custom tools (além do core FS)** | — | 5 | Ilimitado |
| **Connectors Premium (Jira, ADO, Slack)** | — | R$ 199/mês cada | Incluído (até 5) |
| **SIEM Streaming** | — | R$ 299/mês | Incluído |
| **Custom Domain + SSL** | R$ 49/mês | Incluído | Incluído |
| **SSO/SAML/SCIM** | — | R$ 299/mês | Incluído |
| **SLA** | 99.5% | 99.9% | 99.95% |

### Métricas de Uso (Metering Events - Freshservice)

| Evento | Propriedades | Frequência |
|--------|-------------|------------|
| `tool.call` | org_id, env_id, server_id, tool_name (ticket.create, workflow.trigger...), args_hash, latency_ms, status, fs_domain | Por invocação |
| `tool.call.batch` | org_id, env_id, server_id, count, total_latency_ms | Agregado/minuto |
| `freshservice.oauth_refresh` | org_id, env_id, user_id, domain, success | Por refresh |
| `freshservice.rate_limit_hit` | org_id, env_id, endpoint, retry_after_ms | Por evento 429 |
| `env.provisioned` | org_id, env_id, template (freshservice-core), domain | Por provisionamento |
| `audit.export` | org_id, format, size_bytes | Por exportação |

---

## 11. Autenticação, SSO & Tokens — OAuth 2.1 + PKCE (Freshservice)

### Arquitetura de Auth (Freshservice MCP)

```
MCP CLIENT (Host App)
Claude Desktop / Cursor / Custom Agent
         │
         │ Streamable HTTP + Bearer Token
         ▼
MCP GATEWAY (Obot)
• Valida JWT (JWKS cache 5min)
• Resolve org_id + env_id
• Busca Freshservice OAuth tokens no Vault (path por env)
• Injeta Authorization: Bearer <fs_access_token>
• Rate limit por org/env/tool
         │
         │ Freshservice API calls
         ▼
FRESHSERVICE MCP SERVER (Nosso Código)
• Implementa tools: ticket.*, asset.*, catalog.*, change.*, workflow.*
• Chama Freshservice REST API v2
• Trata paginação, erros, retry, webhook events
         │
         │ OAuth 2.0 / API Key
         ▼
FRESHSERVICE (SaaS)
Domínio do tenant: empresa.freshservice.com
OAuth App configurado no Admin Console Freshservice
```

### Fluxo OAuth 2.1 + PKCE — Freshservice Integration (por Ambiente)

```mermaid
sequenceDiagram
    autonumber
    actor Admin
    participant Console as Admin Console
    participant GW as MCP Gateway (Obot)
    participant Vault as Token Vault
    participant FS as Freshservice (OAuth Server)
    
    Admin->>Console: Cadastra domínio Freshservice + OAuth App credentials (para sandbox OU prod)
    Console->>GW: POST /organizations/{org_id}/environments/{env_id}/freshservice/config {domain, client_id, client_secret}
    GW->>Vault: Armazena client_secret criptografado em secret/org/{org_id}/{env_id}/fs-oauth
    GW->>FS: Valida credenciais (GET /api/v2/tickets?per_page=1)
    FS-->>GW: 200 OK
    GW-->>Console: Configuração validada
    
    actor User
    participant App as MCP Host App
    participant Browser
    participant AS as Freshservice OAuth (Authorization Server)
    
    User->>App: Inicia agente / acessa recurso (header X-Environment-ID: sandbox)
    App->>Browser: Redirect → GW /authorize?client_id&redirect_uri&scope&code_challenge&env=sandbox
    Browser->>GW: GET /authorize (valida PKCE, org+env context)
    GW->>AS: Redirect /oauth/authorize (Freshservice OAuth)
    AS->>Browser: Login Freshservice (SSO/SAML/Google/Microsoft) + Consent
    Browser->>AS: Credenciais / MFA
    AS->>Browser: Redirect → GW /callback?code=...
    GW->>AS: POST /oauth/token (code + code_verifier + client_secret)
    AS->>GW: {access_token, refresh_token, expires_in, scope}
    GW->>Vault: Armazena refresh_token criptografado (per-org/env/user)
    GW->>Browser: Redirect → App /callback?code=...
    Browser->>App: GET /callback?code=...
    App->>GW: POST /token (code + code_verifier + client_secret)
    GW->>App: {access_token, refresh_token, expires_in}
    
    loop Tool Calls (ex: workflow.trigger)
        App->>GW: MCP tools/call + Bearer token + X-Environment-ID
        GW->>GW: Valida JWT, resolve org+env
        GW->>Vault: Busca refresh_token Freshservice do path do env
        GW->>FS: POST /oauth/token (refresh_token) → novo access_token
        GW->>FS: POST /api/v2/workflows/123/trigger (com access_token FS)
        FS-->>GW: Execution ID
        GW-->>App: Resultado MCP estruturado
    end
```

### Estrutura de Tokens (JWT) — Access Token Exemplo

```json
{
  "header": { "alg": "RS256", "typ": "JWT", "kid": "gw-key-2024" },
  "payload": {
    "iss": "https://auth.mcp.example/realms/org-acme",
    "sub": "u_abc123",
    "aud": ["mcp-gateway.example", "client_abc"],
    "azp": "client_abc",
    "sid": "sess_xyz789",
    "org_id": "org_acme",
    "env_id": "sandbox",
    "freshservice_domain": "acme-sandbox.freshservice.com",
    "roles": ["admin", "mcp_operator", "fs_agent"],
    "permissions": ["tools:call", "servers:read", "freshservice:tickets:write", "freshservice:workflows:execute"],
    "scope": "openid profile email mcp:tools mcp:servers freshservice:api",
    "auth_time": 1729872000,
    "acr": "mfa",
    "amr": ["pwd", "totp"],
    "exp": 1729875600,
    "iat": 1729872000,
    "jti": "tok_abcdef123456"
  }
}
```

### Rotação & Revogação (Freshservice OAuth)

| Token | TTL Padrão | Renovação | Revogação |
|-------|------------|-----------|-----------|
| **MCP Access Token** | 15 min | Refresh token | Introspection / JWKS cache (5 min) |
| **MCP Refresh Token** | 30 dias (rotativo) | Rotation a cada uso + reuse detection | Revogação imediata (logout global, admin, segurança) |
| **Freshservice Access Token** | 1h (padrão FS) | Proativo (background job 10min antes expiry) | Revogação no Freshservice + purge no Vault |
| **Freshservice Refresh Token** | Não expira (até revogar) | N/A | `DELETE /organizations/{org_id}/environments/{env_id}/freshservice/tokens` + revoke no FS |
| **Session (server-side)** | 30 dias idle | Sliding window | `POST /admin/organizations/{org_id}/sessions/revoke` |

### Multi-Tenant Auth Patterns (Freshservice Context)

| Pattern | Descrição | Caso de Uso Freshservice |
|---------|-----------|--------------------------|
| **Per-User OAuth** | Cada usuário autoriza suas credenciais Freshservice. Tokens criptografados no Vault per-org/env. | Agentes de suporte usando IA para criar/atualizar tickets **em seu nome**. |
| **Shared Service Account** | Admin cadastra 1 service account (API Key ou OAuth client credentials). Gateway controla RBAC. | Automações backend, relatórios, sync de CMDB — **sem usuário humano**. |
| **Hybrid** | Mix: per-user para tickets (auditoria), shared para assets/CMDB/workflows (operações). | **Maioria dos casos enterprise ITSM**. |
| **On-Behalf-Of (OBO)** | Gateway troca token do usuário por token Freshservice via Token Exchange (RFC 8693). | Zero-trust, auditoria completa: "Usuário X disparou workflow via IA às 14:32". |
| **Virtual MCP Bundles** | Agrupa tools Freshservice + conectores (Jira, Slack) sob 1 endpoint + policy única. | Cliente vê 1 servidor MCP "ITSM Unificado". |

---

## 12. Isolamento de Dados & Segurança (Freshservice Focus)

```mermaid
flowchart TB
    subgraph Network["Network"]
        FW[WAF / Cloudflare / AWS Shield]
        VPC[VPC Dedicado por Tier<br/>Enterprise+]
        SG[Security Groups<br/>Deny-by-default]
        MTLS[mTLS CP ↔ DP<br/>SPIFFE/SPIRE]
    end

    subgraph Data["Data"]
        RLS[Row Level Security<br/>PostgreSQL policies por org_id+env_id]
        FS_TOK[Freshservice Tokens Isolation<br/>Vault Namespace/Path per org/env]
        ENC[Encryption at Rest<br/>AES-256 + KMS (BYOK opcional)]
        TLS[TLS 1.3 everywhere]
        BKP[Backups criptografados<br/>RPO 1h / RTO 4h]
        MASK[Data Masking<br/>PII em tickets/logs (LGPD)]
    end

    subgraph Runtime["Runtime"]
        NS[Namespace K8s por org/env<br/>Resource quotas + NetworkPolicy]
        PSP[Pod Security Standards<br/>Restricted]
        SA[Service Accounts<br/>Workload Identity / IRSA]
        SCOPE[Tool-level Scope Enforcement<br/>OPA sidecar: ticket.create vs ticket.read]
    end

    subgraph Operational["Operational"]
        AUD[Audit Logs imutáveis<br/>CloudTrail / Loki + S3 WORM]
        SIEM[Streaming para SIEM cliente<br/>(Splunk, Datadog, S3, Elastic)]
        INC[Incident Response<br/>Runbooks + On-call]
        SCAN[Security Scanning<br/>FS MCP Server image + deps]
        FS_AUDIT[Freshservice Audit Log Sync<br/>Correlação MCP ↔ FS nativo]
    end

    FW --> VPC
    VPC --> SG
    SG --> MTLS
    MTLS --> RLS
    RLS --> FS_TOK
    FS_TOK --> ENC
    ENC --> TLS
    TLS --> BKP
    NS --> PSP
    PSP --> SA
    SA --> SCOPE
    MASK --> SCOPE
    AUD --> SIEM
    SIEM --> INC
    SCAN --> INC
    FS_AUDIT --> INC
```

### Checklist de Conformidade (LGPD / SOC 2 Type II - Freshservice)

- [ ] **Data Processing Addendum (DPA)** padrão + sub-processadores (Freshservice, Cloud, Vault) listados
- [ ] **Right to be forgotten** → API `DELETE /organizations/{org_id}/users/{userId}` + purge tokens Freshservice + job 30d
- [ ] **Data Portability** → Export JSON/CSV completo (configs, audit logs, tokens metadata) via Admin Console ou API
- [ ] **Breach Notification** < 72h (processo documentado + templates) — inclui vazamento de tokens Freshservice
- [ ] **Encryption Keys** → AWS KMS / GCP KMS / HashiCorp Vault; rotação 90d; BYOK para Enterprise
- [ ] **Pen Test** anual + bug bounty (HackerOne/Intigriti) — foco em token leakage Freshservice
- [ ] **SOC 2 Type II** → Auditoria anual (Vanta/Drata para automação)
- [ ] **ISO 27001** alinhamento (políticas, gestão de riscos, continuidade)
- [ ] **Data Residency** → Regiões US/EU/BR aplicáveis via tier; tokens Freshservice nunca saem da região
- [ ] **Freshservice API Compliance** → Respeito a ToS, rate limits, data retention policies do Freshservice

---

## 13. Bases Técnicas Recomendadas (Gateway/Runtime para Freshservice MCP)

| Critério | **Obot (Open Source)** | **MintMCP** | **Scalekit** | **Custom (Go + MCP SDK)** |
|----------|------------------------|-------------|--------------|---------------------------|
| **Licença** | **MIT (Open Source)** | Proprietário | Proprietário | Própria |
| **Arquitetura** | K8s-native (Go + React) | Gateway gerenciado (K8s) | Gateway gerenciado + Auth stack | Total controle |
| **Multi-tenancy** | Namespace K8s + Vault隔离 | Virtual MCP Bundles + TBAC | Per-tenant isolation + Org model | Custom (RLS/Namespace) |
| **Auth** | OIDC/OAuth2, device auth | OAuth 2.0/2.1, SAML, OIDC, OBO | OAuth 2.1, PKCE, CIMD, DCR, XAA | Custom (qualquer) |
| **Registry/Catalog** | ✅ MCP + Skills Registry | ❌ (não tem) | ❌ (usa Smithery/outros) | Custom |
| **STDIO Support** | ✅ Nativo (container sibling) | ✅ OAuth brokering | ✅ Via gateway | Custom |
| **Observabilidade** | Audit logs + Metrics | Agent Monitor (real-time) | Logs + Metrics | Custom (OpenTelemetry) |
| **Freshservice Fit** | **Alto** — custom server como "Skill", gateway unificado | Médio — precisa wrapper | Médio — auth-first | **Máximo** — código próprio |
| **Deploy** | **Self-hosted K8s** (Community) / Hosted (Cloud) | SaaS + Self-hosted (Enterprise) | SaaS + Private/self-hosted | Qualquer |
| **Preço** | **Free (open source)** + Hosted paid | Enterprise sales-only | Free 5 users → $99/mo → $10/user | Custo infra + dev |
| **Ideal para** | **Data sovereignty, self-host, auditável, custom tools** | Regulated enterprise, governance-first | Dev-focused, auth-first, quick start | Requisitos únicos, IP próprio, performance extrema |

### Minha Recomendação: **Obot (Open Source) + Control Plane Próprio + Freshservice MCP Server Custom**

> - **MIT License** → zero vendor lock-in, auditable, customizável para tools Freshservice específicas
> - **K8s-native** → isolamento nativo via namespaces, NetworkPolicy, ResourceQuota por org/env
> - **MCP + Skills Registry built-in** → catálogo privado de versões do Freshservice MCP Server (e futuros)
> - **Gateway unificado (MCP + LLM)** → single endpoint para clientes
> - **Auth flexível** → OIDC/OAuth2, device flow, upstream credential isolation (Vault path per org/env)
> - **Freshservice MCP Server** → **nosso código** (Python/Go) como "Skill" no Obot ou container sidecar
> - **Roadmap**: oferecer **Obot Community + FS MCP Server** no tier Starter (self-service deploy via API) e **Obot Cloud/Enterprise + FS MCP Server gerenciado** nos tiers superiores (managed + SLA + features avançadas + connectors premium)

---

## 14. Registry & Catalog Strategy (Freshservice MCP Server Versions)

```mermaid
flowchart LR
    subgraph Public["Public Registry"]
        OR[Official MCP Registry]
        SM[Smithery.ai]
        GL[Glama.ai]
    end

    subgraph Private["Private Registry (Nossa Plataforma)"]
        PR[Private Registry API<br/>(PostgreSQL + pgvector)]
        SC[Security Scanner<br/>(Syft/Grype + SBOM)]
        SG[Signing & Attestation<br/>(Cosign + Sigstore)]
        POL[Policy Engine<br/>(Allow/Deny/Quarantine)]
    end

    subgraph Tenant["Tenant Catalog"]
        TC[Tenant Allowlist<br/>(Versões FS MCP Server aprovadas)]
        CUST[Custom Tools<br/>(Built by tenant: workflows custom)]
        VER[Version Pinning<br/>(Immutable refs: sha256)]
    end

    OR --> PR
    SM --> PR
    GL --> PR
    PR --> SC
    SC --> SG
    SG --> POL
    POL --> TC
    CUST --> TC
    TC --> VER
```

### Versionamento do Freshservice MCP Server

| Versão | Tools Incluídas | Freshservice API Version | Breaking Changes | Status |
|--------|-----------------|-------------------------|------------------|--------|
| v1.0.0 | tickets.*, assets.*, catalog.* | v2 | — | Stable |
| v1.1.0 | + changes.*, kb.* | v2 | — | Stable |
| v1.2.0 | + cmdb.*, workflows.* | v2 | — | Beta |
| v2.0.0 | Refatoração tools, novos parâmetros | v2 | Sim (inputSchema) | Planejado |

---

## 15. Roadmap de Implementação (16 Semanas - Freshservice MCP Platform)

```mermaid
gantt
    title Roadmap Freshservice MCP Platform - 16 Semanas
    dateFormat  YYYY-MM-DD
    axisFormat  %d/%m
    todayMarker stroke-width:2,stroke:red

    section Fundação (VPS)
    Infra Base (Docker Compose: PG, Redis, Kafka, Vault, Obs)     :a1, 2025-02-03, 14d
    Control Plane API (CRUD Orgs, Envs, Plans, FS Config)         :a2, after a1, 14d
    Provisioning Engine (Temporal + Obot Skill Register)          :a3, after a2, 21d

    section Auth & Gateway
    Obot Standalone Deploy (Docker Compose, config via API)       :b1, 2025-02-03, 14d
    OAuth 2.1 + PKCE Implementation (Auth Server + PRM)           :b2, after b1, 14d
    Multi-env Token Vault (Vault paths per org/env)               :b3, after b2, 10d
    SSO/SAML/SCIM Integration (Keycloak/Scalekit/Ory)             :b4, after b3, 10d

    section Freshservice MCP Server
    FS MCP Server Core (tickets, assets, catalog)                 :fs1, after a1, 21d
    FS MCP Server Extended (changes, kb, cmdb, workflows)         :fs2, after fs1, 14d
    FS OAuth Integration (refresh, rate limit handling, retry)    :fs3, after fs2, 10d
    FS MCP Server Tests + Load Test (500 RPS target)              :fs4, after fs3, 7d

    section Registry & Catalog
    Private Registry API (Metadata, Search, Vector)               :c1, after a2, 14d
    Security Scanner Pipeline (CI/CD + Admission Controller)      :c2, after c1, 10d
    Signing & Attestation (Cosign + Policy)                       :c3, after c2, 7d
    Tenant Catalog UI (Admin Console + CLI)                       :c4, after c3, 10d

    section Billing & Ops
    Stripe Integration (Subs, Metering, Portal, x402)             :d1, after a3, 14d
    Usage Pipeline (Kafka → ClickHouse → Stripe Meter)            :d2, after d1, 10d
    Admin Console (React, RBAC, Audit Logs, Billing)              :d3, after d1, 21d

    section Hardening
    mTLS, Network Policies, WAF, Pen Test                         :e1, after b4, 14d
    Disaster Recovery Drill (RPO/RTO, Chaos Engineering)          :e2, after e1, 10d
    Docs SDKs (TS, Python, Go) + Onboarding Flow                  :e3, after d3, 10d

    section Launch
    Beta Privado (5-10 design partners Freshservice)              :f1, after e2, 14d
    GA + Public Pricing Page + Marketplace                        :f2, after f1, 14d
```

### Marcos & Entregáveis (Freshservice)

| Semana | Marco | Entregável | Critério de Sucesso |
|--------|-------|------------|---------------------|
| 2 | **Infra Pronta (VPS)** | Docker Compose: PG + Redis + Kafka + Vault + Grafana/Loki/Tempo | `docker compose ps` healthy, backups testados |
| 4 | **Control Plane API** | CRUD Orgs/Envs/Plans + Provisioning Engine (Temporal) | Cria org+2 envs em < 60s via API |
| 6 | **FS MCP Server Core** | Tools tickets.*, assets.*, catalog.* + OAuth refresh automático | 100% tests pass, latência p95 < 200ms vs FS API |
| 8 | **Obot + Skill Register** | Obot expõe endpoint por env; skill registration automatizado | 10 tenants de teste provisionados end-to-end |
| 10 | **Auth Funcional** | OAuth 2.1 + PKCE + PRM + JWKS + token refresh FS + revogação | Cypress suite > 90% pass, latência p95 < 150ms |
| 12 | **Registry Live** | Private registry com scan, sign, policy, tenant allowlist | 5 versões FS MCP Server publicadas, 0 vuln críticas |
| 14 | **Billing Live** | Stripe cobra base + overage tool-calls (por tool FS/env) + portal | Fatura correta para 10 tenants simulados 30 dias |
| 16 | **Hardened (VPS)** | mTLS CP↔DP, WAF, Pen Test limpo, DR validado | Relatório pen test sem críticos, RTO < 4h |
| 20 | **Beta → GA** | 5 design partners FS em produção, docs SDKs, pricing público | NPS > 45, churn 0, MRR target atingido |
| 24+ | **K8s Migration** | Helm charts, ArgoCD, HA Vault, Cloud SQL, Ingress | Zero-downtime migration, custos < 30% aumento |

---

## 16. Referências & Fontes (Freshservice + MCP)

### Especificação & Documentação Oficial
- [Model Context Protocol Specification](https://modelcontextprotocol.io/docs/2026-07-28/learn/architecture) — Arquitetura, transports, auth
- [MCP Authorization Spec](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/authorization) — OAuth 2.1, PKCE, PRM
- [Freshservice API v2 Documentation](https://api.freshservice.com/#introduction) — REST API, OAuth, rate limits, webhooks
- [Freshservice OAuth 2.0 Guide](https://developers.freshservice.com/oauth/) — Configuração OAuth App, scopes, refresh tokens
- [Freshservice Workflow Engine API](https://developers.freshservice.com/api/workflow_engine/) — Workflows, execuções, triggers

### Plataformas & Gateways (Análise Comparativa)
- [Obot Platform](https://docs.obot.ai/concepts/architecture) — Open source, K8s-native, MCP + Skills registry
- [MintMCP Authentication Models](https://www.mintmcp.com/docs/authentication-models) — Per-user, shared, hybrid, OBO
- [Scalekit MCP Auth](https://www.scalekit.com/blog-category/mcp-authentication) — CIMD, DCR, XAA, enterprise SSO
- [TrueFoundry MCP Gateway](https://www.truefoundry.com/docs/ai-gateway/mcp/mcp-overview) — Registry, virtual servers, guardrails
- [Composio MCP Authentication](https://composio.dev/content/mcp-authentication-enterprise-teams) — Per-user OAuth, vault, refresh

### Artigos Técnicos & Benchmarks
- [Stack Overflow: Auth in MCP](https://stackoverflow.blog/2026/01/21/is-that-allowed-authentication-and-authorization-in-model-context-protocol) — STDIO vs Streamable HTTP
- [Protecting MCP with OAuth 2.1 (Go + Keycloak)](https://medium.com/@wadahiro/protecting-mcp-server-with-oauth-2-1-a-practical-guide-using-go-and-keycloak-7544eb5379d3)
- [MCP Server Billing Models](https://usagebox.com/articles/how-to-charge-for-mcp-server-2026-per-call-subscription-x402) — Per-call, subscription, freemium, outcome
- [Hosted MCP Platforms Comparison](https://www.merge.dev/blog/hosted-mcp-platforms) — Arcade, Composio, etc.

### Compliance & Security
- [LGPD Guia Prático (ANPD)](https://www.gov.br/anpd/pt-br/assuntos/guias-e-cartilhas)
- [SOC 2 Type II Checklist](https://www.vanta.com/soc-2-checklist)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [RFC 9728 - OAuth 2.0 Protected Resource Metadata](https://datatracker.ietf.org/doc/html/rfc9728)
- [RFC 8693 - OAuth 2.0 Token Exchange (OBO)](https://datatracker.ietf.org/doc/html/rfc8693)

### Ferramentas de Apoio
- [Obot GitHub](https://github.com/obot-platform/obot) — MIT licensed, K8s operator
- [MCP SDKs](https://github.com/modelcontextprotocol) — Python, TypeScript, Go, Rust
- [Temporal](https://temporal.io/) — Provisioning workflows duráveis
- [OpenPolicyAgent](https://www.openpolicyagent.org/) — AuthZ (TBAC/RBAC)
- [Cosign/Sigstore](https://docs.sigstore.dev/) — Signing & verification
- [Lago](https://www.getlago.com/) / [OpenMeter](https://openmeter.io/) — Open-source billing/metering
- [Turborepo](https://turbo.build/repo) / [Nx](https://nx.dev/) — Monorepo tooling

---

## 17. Próximos Passos Imediatos (Esta Semana - Freshservice MCP em VPS)

1. **Provisionar VPS** (Hetzner CX42 / DigitalOcean 8GB / AWS t3.medium) → Docker + Compose (arquivo acima).
2. **Deploy Obot Community** via Docker Compose → validar gateway MCP + Skills Registry + auth flow.
3. **Desenvolver Freshservice MCP Server v1.0.0** (Python + FastMCP ou Go + MCP SDK):
   - Tools: `ticket.create`, `ticket.get`, `ticket.list`, `ticket.update`, `asset.create`, `asset.list`, `service_catalog.list`, `workflow.list`, `workflow.trigger`, `workflow.get_execution`
   - OAuth 2.0 client com refresh token automático + rate limit handling (429 retry com backoff)
   - Containerizado, health checks, structured logging (JSON), métricas Prometheus
4. **Escrever Provisioning Engine** (Go/TS + Temporal) que: cria schema PG `org_{id}_{env}`, cria path Vault `secret/org/{id}/{env}/fs-oauth`, registra Skill + SkillInstance no Obot via API, configura nginx subdomínio (`{org}-{env}.mcp.seudominio.com`), emite `environment.ready` webhook.
5. **Definir schema de billing** no Stripe (products, prices, meters `tool_calls` com labels `tool_name=ticket.create|workflow.trigger...`, `org_id`, `env_id`) + webhook `invoice.payment_succeeded` → libera features/quotas.
6. **Montar repo `mcp-control-plane`** (monorepo: `apps/control-plane-api`, `apps/admin-console`, `apps/provisioning-engine`, `packages/shared-types`, `infra/`) com CI/CD (GitHub Actions + Turborepo).
7. **Criar repos `fs-mcp-server`** (independente) com CI que publica imagem + `mcp-manifest.json` assinado no Registry privado.
8. **Agendar 2h com 3-5 design partners** (MSPs, consultorias ITSM, TI enterprise) para validar: pricing, auth requirements (SSO/SAML/SCIM), data residency, connector wishlist (Jira, Azure DevOps, Slack, Teams), SLA expectations, ferramentas Freshservice mais usadas.
9. **Prototipar 2 MCP Servers custom adicionais** (ex: PostgreSQL readonly para relatórios, n8n workflow executor) em repos próprios para testar provisioning + registry + gateway end-to-end além do core Freshservice.

---

*Documento gerado em 2025-01-25 — Versão 2.0 (Freshservice Focus + VPS + Multi-MCP Strategy)*  
*Próxima revisão: após validação com design partners (Semana 4)*
