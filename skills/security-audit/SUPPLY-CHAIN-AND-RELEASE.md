# Caça em Cadeia de Suprimentos e Release (`Supply Chain and Release Hunting`)

#### When to use this file

Utilize este guia quando o repositório resolver dependências, compilar contribuições externas, executar pipelines de CI/CD, gerar artefatos de release, assinar ou promover builds, carregar plugins/extensões ou atualizar softwares implantados. Este domínio audita as passagens de confiança desde o código-fonte e pacotes externos até o artefato binário/contêiner executado pelo usuário (veja também controles `AI04` e `INF03`). Use `MEMORY-SAFETY-AND-BINARY.md` para falhas internas de loaders binários e `CLOUD-AND-DEPLOYMENT.md` para permissões de runtime em nuvem.

## Core discipline (incluir em todo prompt de agente para este domínio)

```
- Uma dependência mutável ou com CVE conhecida não é automaticamente um achado confirmado sozinha: demonstre quem pode influenciar a resolução, qual build a consome e qual fronteira de execução ou release é alcançada.
- Siga a integridade em cada passagem de bastão (handoff): identidade do código-fonte, entradas resolvidas, runner de build, identidade do artefato, resultado de testes, assinatura/atestação SLSA, promoção de ambiente e consumidor de atualização.
- Configuração de CI/CD é código de autorização. Estabeleça qual evento disparou o workflow, o código de quem está rodando, quais segredos e tokens estão acessíveis e o que ele pode publicar ou alterar.
- Um checksum baixado do mesmo local não confiável que o próprio artefato não estabelece integridade independente: identifique a raiz de confiança (trust root) e o comportamento em falha.
- Use `confirmed` para falhas de controle contidas no repositório com validação local segura. Use `needs_validation` para regras de proteção de branch, runners hospedados, registries externos ou políticas de assinatura invisíveis no código.
```

## Dependency and build-input attack classes (subagent_type: `general`)

**Dependency source and namespace confusion**
Configurações de gerenciadores de pacotes (`npm`, `pip`, `cargo`, `go`, `maven`) capazes de selecionar um pacote público em vez de privado (*dependency confusion*), registry espelho inseguro ou URL alternativa. Analise escopos, prioridade de fontes, uso de lockfiles (`--frozen-lockfile` / `npm ci`), checksums e scripts de ciclo de vida (`postinstall`).

**Mutable and unbound build inputs**
Builds que consomem branches móveis, tags Git mutáveis, submódulos não verificados, binários baixados via `curl | sh`, GitHub Actions sem pinning por hash SHA-256 (`@main` ou `@v4` em vez de SHA imutável), tags de contêiner `:latest` ou pesos de modelos de IA não verificados (`AI04`).

**Generated-source and codegen provenance gaps**
Schemas externos, clientes gerados automaticamente, arquivos vendored, traduções ou blobs binários que produzem código executável ou embarcado sem passar pelos mesmos gates de revisão e integridade do código principal.

**Build-context inclusion**
Segredos, arquivos `.env`, metadados `.git`, chaves de teste ou artefatos locais copiados acidentalmente para dentro de imagens Docker (`COPY . .` sem `.dockerignore` estrito) ou pacotes publicados em registries públicos.

## CI and automation attack classes (subagent_type: `general`)

**Untrusted code in a privileged workflow**
Pull requests de forks externos, comentários em issues ou eventos automatizados que executam código controlado pelo contribuidor com acesso a segredos protegidos, `GITHUB_TOKEN` com permissão de escrita, credenciais OIDC de nuvem ou runners auto-hospedados persistentes (ex.: uso inseguro de `pull_request_target` fazendo checkout do commit do PR — *Poisoned Pipeline Execution*).

**Workflow command and expression confusion**
Nomes de branches, títulos de PR, mensagens de commit, corpos de issues, valores de matrix ou nomes de artefatos controlados pelo atacante interpolados diretamente em blocos `run:` de shell (ex.: `${{ github.event.pull_request.title }}`) ou expressões de template privilegiadas sem passagem segura por variáveis de ambiente.

**Cache, artifact, and workspace trust mixing**
Um job de menor confiança popula cache de dependências, artefato intermediário de build ou diretório compartilhado que um job posterior de maior privilégio (release/deploy) restaura e executa ou publica sem verificação criptográfica de digest.

**Automation identity overreach**
Jobs de CI recebendo permissões amplas demais (`permissions: write-all`, roles OIDC sem restrição de branch/environment) onde entradas não confiáveis conseguem selecionar o recurso afetado.

## Release and update attack classes (subagent_type: `general`)

**Build-to-promotion substitution**
Testes, revisão, assinatura (Cosign/SLSA) e publicação referenciando tags mutáveis ou nomes de arquivos em vez do mesmo digest SHA-256 imutável entre o momento do build e o momento do release (*TOCTOU de artefato*).

**Release authorization and signing-policy gaps**
Assinaturas ou atestações aceitas pelo consumidor sem validar identidade do workflow emissor, repositório, branch, ambiente ou expiração/revogação da chave (verificar apenas "assinatura criptograficamente válida" sem validar *quem* assinou).

**Update metadata and rollback confusion**
Atualizadores automáticos que validam os bytes do pacote, mas não vinculam versão, produto, plataforma, canal, caminho de destino ou proteção contra *rollback/freeze attack*, ou que aceitam metadados e payload de transações distintas.

**Plugin and extension trust expansion**
Pacotes de extensão/plugins ou servidores MCP de terceiros que adquirem autoridade no host além do escopo declarado, ou cujos hooks de instalação executam antes da verificação de autenticidade e assinatura.

## Universal moves (aplicar transversalmente neste domínio)

- Caminhe de trás para frente a partir do artefato publicado ou atualização instalada até cada código-fonte, entrada gerada, credencial, runner, cache, teste e decisão de autorização.
- Compare lado a lado workflows disparados por eventos não confiáveis (`pull_request`, `issue_comment`) e eventos protegidos (`push` em `main`, `release`), inspecionando todo canal persistido (cache/artifacts) que cruza entre eles.
- Revise caminhos de chave revogada, falha de download, ausência de atestação SLSA e rollback parcial: a política de falha deve ser sempre *fail-closed*.

## Validation rules (aplicar antes de reportar QUALQUER achado aqui)

1. Nomeie o ator de menor confiança, o código/cache/artefato/metadado controlável, o job ou atualizador confiável consumidor e a publicação não autorizada, vazamento de segredo ou execução privilegiada resultante.
2. Comprove a quebra de identidade do artefato na passagem de bastão: um nome mutável ou digest desvinculado deve alcançar um consumidor real.
3. Verifique os padrões da versão exata do gerenciador de pacotes, plataforma Git e ferramenta de assinatura; controles hospedados externos não visíveis no repositório exigem `needs_validation`.
4. Mantenha toda validação local restrita a repositórios/fixtures descartáveis e marcadores fictícios, jamais publicando pacotes ou alterando releases reais.
5. Retorne `confirmed` apenas com cadeia completa visível no código e impacto real demonstrado; retorne `needs_validation` detalhando a regra de branch protection, runner, registry ou assinatura que o proprietário deve inspecionar.
