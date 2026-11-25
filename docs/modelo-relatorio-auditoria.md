# Modelo de Relatório de Auditoria de Segurança Técnica (v3.0)

> **Projeto / Repositório:** `[Nome do Projeto]`  
> **Auditor / Orquestrador:** `[Nome do Auditor ou Agente Coordenador]`  
> **Data:** `[DD/MM/AAAA - HH:MM]`  
> **Source Ref (Commit Auditado):** `[Hash Git inicial / estado da worktree]`  
> **Perfil e Escopo:** `[ ] quick  [x] standard  [ ] deep` • `Política: sandboxed-source-and-local-only`  
> **Validadores Executados:** `validate-coverage-ledger.cjs (PASS)` • `validate-findings.cjs (PASS)`

---

## 1. Resumo Executivo e Cobertura

- **Postura Geral de Segurança:** `[Síntese objetiva de 1 parágrafo sobre as fronteiras e controles observados]`
- **Unidades do Coverage Ledger:** `Total: [N] | Covered: [X] | Candidate: [Y] | Blocked: [Z] | Deferred / Out-of-Scope: [W]`
- **Controles do Framework Avaliados:** `60 / 60`
- **Vulnerabilidades Confirmadas (`confirmed`):** `[X]`
  - `critical`: `[0]` • `high`: `[1]` • `medium`: `[0]` • `low`: `[0]` • `informational`: `[0]`
- **Hipóteses Pendentes de Validação (`needs_validation` — sem severidade):** `[X]`
- **Candidatos Refutados na Validação Adversarial (`rejected`):** `[X]`

---

## 2. Achados Confirmados (`confirmed`)

| Fingerprint | Severidade (`overall <= impact`) | Título | Fronteira Violada | Resultado Local Observado |
| :--- | :---: | :--- | :--- | :--- |
| `api-orders-missing-tenant-scope` | `high` | BOLA na consulta de pedidos | `src/orders/controller.ts#getOrder` | Conta fictícia do Tenant A leu o pedido fictício `#50` do Tenant B |

### Detalhamento: `api-orders-missing-tenant-scope` — BOLA na consulta de pedidos

- **Controle Correlato:** `APP03` (BOLA / IDOR) & `DATA-ISOLATION-AND-LIFECYCLE.md`
- **Severidade:** `overall_severity: high` (`likelihood: high`, `impact: high`) • **Confiança:** `high`
- **Principal de Menor Confiança:** Usuário autenticado comum pertencente ao Tenant A.
- **Trace no Código-Fonte (`entrypoint → propagation → sink`):**
  1. `[entrypoint]` `src/orders/routes.ts:18` — Rota `GET /api/v1/orders/:id` recebe `req.params.id`.
  2. `[propagation]` `src/orders/controller.ts:45` — Repassa `id` sem extrair `req.user.tenantId`.
  3. `[sink]` `src/orders/repository.ts:29` — Executa `SELECT * FROM orders WHERE id = $1`.

#### Reprodução Local Delimitada (Sandbox)
- **Entrada / Fixture:** Dois tenants fictícios (`tenant-a`, `tenant-b`); requisição autenticada como `tenant-a` para `/api/v1/orders/50` (registro criado para `tenant-b`).
- **Resultado Observado:** HTTP `200 OK` retornando os dados completos do pedido pertencente ao `tenant-b`.

#### Menor Correção Eficaz (Camada Autoritativa)
```typescript
export async function getOrder(req: Request, res: Response) {
    const tenantId = req.user.tenantId;
    const order = await db.query(
        'SELECT * FROM orders WHERE id = $1 AND tenant_id = $2',
        [req.params.id, tenantId]
    );
    if (!order.rows[0]) {
        return res.status(404).json({ error: 'Pedido não encontrado' });
    }
    return res.json(order.rows[0]);
}
```

#### Testes de Validação Executados
- **Negativo (Adversarial):** `tenant-a` consulta pedido de `tenant-b` → `404 Not Found`.
- **Positivo (Legítimo):** `tenant-b` consulta próprio pedido → `200 OK`.

---

## 3. Hipóteses Bloqueadas (`NEEDS VALIDATION` — Sem Severidade)

> Registros abaixo possuem caminho real no código, mas dependem de fatos de implantação externos ao repositório. Não devem receber severidade até confirmação.

| Fingerprint | Trace Inicial | Bloqueador Exato (`blockers`) | Verificação Segura pelo Proprietário (`deployment`) |
| :--- | :--- | :--- | :--- |
| `ingress-forwarded-host-trust` | `src/auth/reset.ts:62` | Configuração do proxy reverso de produção não está neste repositório | Inspecionar config do Cloudflare/ALB/Nginx para confirmar se `X-Forwarded-Host` do cliente é sobrescrito na borda |

---

## 4. Tabela de Cobertura dos 60 Controles

> Estados aceitos: `verificado`, `falha_confirmada`, `suspeita`, `nao_aplicavel`, `bloqueado`.

| Controle | Domínio | Estado | Evidência Sanitizada / Referência |
| :--- | :--- | :---: | :--- |
| **APP01** | Autenticação e Reset de Senha | `verificado` | Argon2id (64 MiB) e token SHA-256 de uso único em `src/auth/` |
| **APP02** | Sessão, JWT e OAuth 2.1 | `verificado` | PKCE `S256` obrigatório e algoritmo fixado em `src/auth/jwt.ts` |
| **APP03** | BOLA / IDOR (OWASP API1) | `falha_confirmada` | Ver achado `api-orders-missing-tenant-scope` |
| **PRO01** | Framing HTTP e Cache | `suspeita` | Ver lead `ingress-forwarded-host-trust` (`needs_validation`) |
| **MEM01** | Segurança de Memória e Binários | `nao_aplicavel` | Aplicação 100% TypeScript sem módulos nativos/FFI próprios |
| ... | *(demais controles até GOV03)* | ... | ... |

---

## 5. Notas de Hardening (Defesa em Profundidade)

- *(Registrar aqui melhorias recomendadas que não constituem vulnerabilidades exploráveis por já possuírem outra camada bloqueando o vetor).*

---

## 6. Parecer de Release

- [ ] **Aprovado para Produção:** Zero vulnerabilidades confirmadas `critical`/`high` abertas e validadores JSON aprovados (`PASS`).
- [ ] **Bloqueado:** Existem falhas confirmadas `critical`/`high` ou validações obrigatórias pendentes.
