# ✅ Checklist Operacional de Auditoria: 60 Controles + Ciclo de 6 Fases (v3.0)

Utilize este checklist em auditorias manuais ou assistidas por agentes de IA. Marque `[x]` conforme cada etapa e controle técnico forem verificados.

---

### Fase 0 — Execução do Pipeline de Auditoria (6 Fases)

- [ ] **Fase 1 (Reconnaissance):** Superfícies, principais, cópias derivadas e fronteiras mapeados em `architecture.md` e `coverage-ledger.json` validado (`validate-coverage-ledger.cjs`).
- [ ] **Fase 2 (Hunting + Coverage Critics):** Ondas de caça executadas por subagentes isolados em sandbox e auditadas por Críticos de Cobertura independentes.
- [ ] **Fase 3 (Adversarial Validation):** Cada candidato revisado por um verificador fresco independente tentando refutá-lo antes da confirmação.
- [ ] **Fase 4 (Structured Output):** `findings.json` gravado separando `confirmed`, `needs_validation` (sem severidade) e `rejected`, aprovado por `validate-findings.cjs`.
- [ ] **Fase 5 (Independent Verification):** Revisão independente linha por linha de todos os registros finais.
- [ ] **Fase 6 (Target-Neutral Reporting):** `REPORT.md`, `FINDINGS-DETAIL.md` e `NEEDS-VALIDATION.md` gerados sem discrepância com o JSON.

---

### 1. Aplicações e APIs (APP01–APP20)

- [ ] **APP01** — Autenticação robusta (Argon2id/bcrypt), proteção contra enumeração e reset de uso único.
- [ ] **APP02** — Sessões e Tokens (OAuth 2.1 com PKCE `S256`, validação estrita de JWT/PASETO, cookies `HttpOnly; Secure; SameSite`).
- [ ] **APP03** — Autorização em Nível de Objeto (BOLA / IDOR — OWASP API1:2023) com validação de tenant no servidor.
- [ ] **APP04** — Autorização de Propriedades e Mass Assignment (OWASP API3:2023) com DTOs e allowlists estritas.
- [ ] **APP05** — Autorização em Nível de Função (BFLA — OWASP API5:2023) em cada rota e método administrativo.
- [ ] **APP06** — Prevenção de Injeções (SQL parametrizado, comandos sem shell `execFile`, NoSQL tipado, indireta/2ª ordem).
- [ ] **APP07** — Cross-Site Scripting (XSS), sanitização contextual de HTML/Markdown (DOMPurify) e CSP com nonces.
- [ ] **APP08** — Proteção contra CSRF (`SameSite` + tokens anti-CSRF) e CORS sem reflexão arbitrária de `Origin`.
- [ ] **APP09** — Server-Side Request Forgery (SSRF — OWASP API7:2023) com bloqueio de IPs privados, IMDS (`169.254.169.254`) e DNS rebinding.
- [ ] **APP10** — Upload Seguro de Arquivos (validação por magic bytes, UUIDs, prevenção de Zip Slip / Path Traversal).
- [ ] **APP11** — Validação Estrita de Schemas e Contratos (Contract-First com Zod/Pydantic/OpenAPI e limites de tamanho/aninhamento).
- [ ] **APP12** — Consumo de Recursos, Rate Limiting e DoS (Token Bucket / Redis, HTTP 429, proteção pré-autenticação).
- [ ] **APP13** — Proteção de Fluxos de Negócio Sensíveis (anti-scalping, anti-hoarding, PoW/Turnstile).
- [ ] **APP14** — Concorrência, Race Conditions e Idempotência (locks transacionais atômicos e `Idempotency-Key`).
- [ ] **APP15** — Gerenciamento de Inventário de APIs (eliminação de Shadow APIs, cabeçalhos `Deprecation`/`Sunset`, paridade de patches).
- [ ] **APP16** — Consumo Seguro de APIs Externas e Webhooks (validação de schema, timeouts curtos e HMAC-SHA256).
- [ ] **APP17** — Hardening de GraphQL (Introspection desativada em prod, limites de profundidade/custo, Trusted/Persisted Queries).
- [ ] **APP18** — Headers HTTP de Segurança (HSTS preload, `nosniff`, `frame-ancestors`, `COOP`/`CORP`, TLS 1.3).
- [ ] **APP19** — Tratamento de Erros e Logs (zero stack traces no cliente, `trace_id` correlacionado, redação automática de PII/segredos).
- [ ] **APP20** — Criptografia em Trânsito Interno (mTLS entre microsserviços e identidades SPIFFE/SPIRE).

---

### 2. Inteligência Artificial, Agentes e RAG (AI01–AI10)

- [ ] **AI01** — Injeção de Prompt Direta e Indireta (delimitadores com nonces, guardrails determinísticos e isolamento de RAG).
- [ ] **AI02** — Divulgação de Informações Sensíveis e PII (sanitização pré-contexto, Zero Data Retention e redação em traces).
- [ ] **AI03** — Quebra da Tríade Letal, HITL com **Action-Confirmation Binding** e consistência entre schema e dispatcher.
- [ ] **AI04** — Segurança da Cadeia de Suprimentos de IA (uso exclusivo de Safetensors, hashes verificados e AI-BOM CycloneDX/SPDX).
- [ ] **AI05** — Prevenção de Envenenamento de Dados, Memória Persistente e RAG (*PoisonedRAG*).
- [ ] **AI06** — Consumo Ilimitado e Denial of Wallet (limites duplos RPM/TPM, circuit breakers em loops de subagentes).
- [ ] **AI07** — Alucinação e Autoridade Indevida (validação determinística fora do modelo, citações obrigatórias e CoVe).
- [ ] **AI08** — Proteção contra Extração de Contexto Sensível (zero credenciais ou regras secretas de autorização no System Prompt).
- [ ] **AI09** — Segurança em Vetores e Embeddings (prevenção de Vec2Text e filtragem CBAC multi-tenant obrigatória no índice).
- [ ] **AI10** — Tratamento Seguro de Saídas de IA (saídas tratadas como 100% não confiáveis antes de HTML, SQL ou comandos).

---

### 3. Bancos de Dados Relacionais, NoSQL e Vetoriais (DB01–DB08)

- [ ] **DB01** — Identidades de Conexão e Menor Privilégio (separação estrita entre DDL de migração, DML de runtime e leitura IA).
- [ ] **DB02** — PostgreSQL Row-Level Security (`ENABLE` + `FORCE ROW LEVEL SECURITY` com políticas restritivas por sessão).
- [ ] **DB03** — Criptografia em Repouso (TDE), em Trânsito (`sslmode=verify-full`) e em Nível de Coluna (CLE / AES-256-GCM).
- [ ] **DB04** — Connection Pooling Seguro (PgBouncer com `DISCARD ALL` e autenticação `scram-sha-256`).
- [ ] **DB05** — Prevenção de SQL Injection e Armadilhas de ORM (prepared statements e allowlist estrita em `ORDER BY`/raw queries).
- [ ] **DB06** — Prevenção de NoSQL Injection (tipagem estrita primitiva, bloqueio de `$ne`/`$gt`/`$where`).
- [ ] **DB07** — Hardening de Bancos Vetoriais (subnets privadas sem exposição pública, auth obrigatória e metadados sem PII).
- [ ] **DB08** — Auditoria de Banco (`pgAudit`) e Regra de Backup 3-2-1-1-0 com cópia imutável WORM e teste de restore.

---

### 4. Integração IA → API → Banco (INT01–INT03)

- [ ] **INT01** — Identidade do Cliente Preservada (backend injeta `user_id`/`tenant_id` autenticados; zero uso de `service_role` livre).
- [ ] **INT02** — Ferramentas de Negócio Atômicas e Queries Mediadas (schemas fechados; AST parser/réplica read-only se Text-to-SQL).
- [ ] **INT03** — Isolamento Estrito de Conversas, Memória Persistente, Cache Semântico e Streaming SSE/WS.

---

### 5. Infraestrutura, DevSecOps e Redes Zero Trust (INF01–INF06)

- [ ] **INF01** — Contêineres Distroless/Mínimos Non-Root, Kubernetes PSS `Restricted` e monitoramento eBPF (Falco).
- [ ] **INF02** — Segurança de IaC e GitOps (bloqueio de configurações inseguras com Checkov/Trivy em PRs).
- [ ] **INF03** — CI/CD Supply Chain e SLSA (sem `pull_request_target` inseguro, pinning SHA-256, assinatura Cosign e OIDC).
- [ ] **INF04** — Gestão Centralizada de Segredos (Vault/Secrets Manager com credenciais dinâmicas efêmeras em `tmpfs`).
- [ ] **INF05** — Redes Zero Trust (subnets em 3 camadas, mTLS `STRICT` via Service Mesh e WAF gerenciado).
- [ ] **INF06** — Governança Cloud CSPM/ASPM e IAM sem curingas (`*`) para identidades humanas e de workloads.

---

### 6. Hardening por Stack (STK01–STK04)

- [ ] **STK01** — WordPress e PHP (XML-RPC desativado, uploads sem execução PHP, `DISALLOW_FILE_EDIT`, `wp-config.php` isolado).
- [ ] **STK02** — React / Next.js (Server Actions protegidas individualmente contra BOLA, zero segredos em `NEXT_PUBLIC_` ou `__NEXT_DATA__`).
- [ ] **STK03** — Node.js / Python / Go (anti-Prototype Pollution, sem `pickle`/`yaml.load` inseguro, propagação de `context.Context`).
- [ ] **STK04** — Model Context Protocol (MCP stdio em sandbox, OAuth 2.1 remoto, vínculo de identidade/request ID e anotações não confiáveis).

---

### 7. Protocolos, Client-Side Avançado, IPC Nativo e Memória (PRO01–PRO03, CLI01, IPC01, MEM01)

- [ ] **PRO01** — Framing HTTP, Prevenção de Request Smuggling, Sanitização `X-Forwarded-*` e Isolamento contra Cache Poisoning/Deception.
- [ ] **PRO02** — Passkeys/WebAuthn (`rpId`/`origin`/`challenge`/`UV`), Step-Up Assurance Binding e Account Linking sem colisão.
- [ ] **PRO03** — Segurança de RPC (gRPC/Protobuf), Filas/Brokers, Isolamento de DLQ, Idempotência e Ordenação de Eventos.
- [ ] **CLI01** — Client-Side Avançado: Prevenção de DOM Clobbering, Allowlist exata em `postMessage`/WS, Higiene de Service Workers e XS-Leaks.
- [ ] **IPC01** — Desktop/Mobile: Universal/App Links verificados, Isolamento de Bridges WebView/Electron e Peer Auth em Sockets/IPC Local.
- [ ] **MEM01** — Segurança de Memória e Binários: Aritmética verificada, Anti-UAF/Double-Fetch, Contratos FFI/ABI seguros e Sanitizers limpos.

---

### 8. Governança, Privacidade e Red Teaming (GOV01–GOV03)

- [ ] **GOV01** — Conformidade LGPD (ANPD 3 dias úteis) e GDPR (72h), ROPA, Propagação de Exclusão/Tombstones em Cópias Derivadas e SDM/DDM.
- [ ] **GOV02** — Plano de Resposta a Incidentes Híbridos de IA (playbooks MITRE ATLAS, trilha forense imutável e *Kill Switch*).
- [ ] **GOV03** — AI Red Teaming Contínuo e Avaliação de Viés em CI/CD (Promptfoo / Garak / PyRIT).
