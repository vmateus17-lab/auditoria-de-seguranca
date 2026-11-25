# Guia de Integração para Agentes e IDEs com IA

Este repositório oferece **duas formas complementares** de equipar agentes de codificação com capacidades avançadas de segurança:

1. **Skill Multi-Agente Instalável (`skills/security-audit/`)**: transforma o agente em um orquestrador de auditoria em 6 fases com validação adversarial e validadores determinísticos JSON.
2. **Framework Mestre de Engenharia Defensiva (`seguranca-hibrida-projetos-ia-api-bancos.md`)**: especifica 60 controles práticos (*Inspecionar → Corrigir → Verificar*) para hardening e remediação direta no código.

---

## 1. Instalação Rápida via Skills CLI (Antigravity, Claude Code, Cursor, Codex, Windsurf)

Instale a skill `security-audit` diretamente deste repositório público usando o [Skills CLI](https://skills.sh):

```bash
# Instalação no projeto atual
npx skills add https://github.com/vmateus17-lab/auditoria-de-seguranca \
  --skill security-audit

# Instalação global para o seu usuário
npx skills add https://github.com/vmateus17-lab/auditoria-de-seguranca \
  --skill security-audit \
  --global
```

Após instalar, basta pedir naturalmente ao seu agente no repositório que deseja auditar:

```text
Execute uma auditoria de segurança completa neste repositório (perfil standard) e salve os artefatos em ~/audits/meu-projeto
```

Ou, para dúvidas focadas (ativa automaticamente o *Guidance Mode* sem criar arquivos):

```text
Revise a segurança das rotas em ./src/api focando em BOLA, sessão e cache
```

---

## 2. Google Antigravity / Claude Code (`CLAUDE.md` ou Skills Locais)

Para projetos que desejam tanto a auditoria estruturada quanto a aplicação contínua dos 60 controles durante o desenvolvimento:

1. Copie `skills/security-audit` para `.agents/skills/security-audit` (ou `~/.gemini/antigravity/skills/security-audit`).
2. No arquivo `CLAUDE.md` ou nas regras do workspace, adicione:

```markdown
## Protocolo de Auditoria e Segurança Defensiva
- Para auditorias completas ou pentests em código, utilize a skill `security-audit` (6 fases com validação adversarial e validadores `.cjs`).
- Ao desenvolver ou corrigir rotas de API, bancos de dados, filas, frontends ou agentes de IA, aplique rigorosamente os 60 controles de `seguranca-hibrida-projetos-ia-api-bancos.md`:
  - Valide BOLA/BFLA no servidor (`APP03`, `APP05`) e schemas fechados (`APP11`).
  - Imponha PostgreSQL `FORCE ROW LEVEL SECURITY` (`DB02`) e prepared statements (`DB05`).
  - Em IA/MCP, aplique isolamento de contexto, Action-Confirmation Binding (`AI03`) e trate saídas/metadados como não confiáveis (`AI10`, `STK04`).
  - Escreva a trinca de testes (negativo, limite e legítimo) para toda correção.
```

---

## 3. Cursor IDE (`.cursor/rules/` ou `.cursorrules`)

Carregue o framework como Regra de Projeto persistente:

```markdown
# Segurança e Auditoria Contínua
Atue como engenheiro e auditor de segurança seguindo os 60 controles de `./seguranca-hibrida-projetos-ia-api-bancos.md` e a disciplina de verificação de `./skills/security-audit/SKILL.md`.

Antes de concluir alterações em autenticação, APIs, bancos, mensageria, client-side ou IA:
1. Confirme o invariante de segurança no último ponto confiável de decisão no servidor.
2. Não exponha segredos, tokens ou PII em logs, bundles SSR ou prompts.
3. Não reporte hipóteses dependentes de deploy externo como falhas confirmadas sem evidência; classifique pendências externas separadamente.
4. Crie testes automatizados de abuso comprovando o bloqueio.
```

---

## 4. GitHub Copilot Chat e Revisão de Pull Requests em CI/CD

Em sessões de revisão de PR ou pipelines automatizados de CI/CD:

```text
@workspace Avalie o diff desta branch usando os controles de seguranca-hibrida-projetos-ia-api-bancos.md e os guias de domínio em skills/security-audit/. Confirme apenas violações reais de fronteira de confiança com arquivo:linha exatos, separe hipóteses dependentes de deploy como needs_validation (sem severidade especulativa) e sugira o menor patch eficaz com teste de regressão.
```
