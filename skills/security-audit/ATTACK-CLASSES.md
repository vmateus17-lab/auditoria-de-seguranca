# Classes de Ataque (`Attack Classes`)

#### Classes de ataque — selecionar e subdividir com base na Fase 1

Selecione as classes de ataque relevantes para o tipo de aplicação auditada. Nem toda classe aplica-se a toda base de código. A lista abaixo é o ponto de partida ordinário; adicione classes específicas identificadas na Fase 1 e divida repositórios grandes por subsistema. Estruture o trabalho para encontrar, validar, corrigir e priorizar vulnerabilidades reais. Mantenha a validação restrita à revisão de código e fixtures locais delimitadas em sandbox; nunca desenvolva cadeias destrutivas pós-exploração, não teste disponibilidade em serviços ativos e não atue em ambientes compartilhados.

Use `confirmed` somente quando a evidência no código-fonte e a validação local delimitada comprovarem a violação completa da fronteira e um resultado relevante. Use `needs_validation` quando um fato específico de implantação, provedor, plataforma, identidade ou runtime estiver indisponível localmente, declarando o fato ausente e o check seguro que o resolve.

> **Alvos nativos / binários / kernel** (C/C++, Rust `unsafe`, módulos de kernel, parsers e decoders, FFI, runtimes concorrentes, loaders binários, JITs, firmware): use as classes de segurança de memória, inteiros/ABI, concorrência, loaders e interfaces privilegiadas em [MEMORY-SAFETY-AND-BINARY.md](MEMORY-SAFETY-AND-BINARY.md) (Controle `MEM01`).
>
> **Alvos de IA / LLM / agentes** (chatbots, pipelines RAG, memória persistente, agentes com *tool calling*, servidores/clientes MCP, montagem de prompts ou ações guiadas por modelo): use as classes de contexto, envenenamento de memória, vinculação de ação (*action-binding*), schemas de ferramentas, identidade MCP e saída em [AI-AND-LLM.md](AI-AND-LLM.md) (Controles `AI01–AI10`, `INT01–INT03`, `STK04`).
>
> **Alvos HTTP, web e identidade** (aplicações web, APIs REST/GraphQL, proxies reversos, CDNs, gateways, parsers HTTP customizados, sessões, CSRF, JWT, OAuth/OIDC, SAML, MFA, passkeys/WebAuthn, recuperação/vinculação de contas, API keys ou mTLS): use [WEB-PROTOCOL-AND-AUTH.md](WEB-PROTOCOL-AND-AUTH.md) (Controles `APP01–APP08`, `APP18`, `APP20`, `PRO01–PRO02`).
>
> **Alvos client-side e navegador** (SPAs, extensões de navegador, webviews embarcadas, service workers, armazenamento no browser, `postMessage`, CORS, WebSockets ou renderização DOM): use [CLIENT-SIDE.md](CLIENT-SIDE.md) (Controles `APP07–APP08`, `STK02`, `CLI01`).
>
> **Alvos de cadeia de suprimentos e release** (resolução de dependências, código gerado, pipelines CI/CD, assinatura/promoção de artefatos, atualizadores automáticos, plugins ou extensões): use [SUPPLY-CHAIN-AND-RELEASE.md](SUPPLY-CHAIN-AND-RELEASE.md) (Controles `AI04`, `INF03`).
>
> **Alvos de nuvem e implantação** (IAM, Infraestrutura como Código, contêineres/Kubernetes, service mesh, serverless/edge, ingress, eventos de provedores ou configuração de runtime): use [CLOUD-AND-DEPLOYMENT.md](CLOUD-AND-DEPLOYMENT.md) (Controles `INF01–INF06`).
>
> **Alvos de protocolos, RPC e mensageria** (gRPC, transportes GraphQL, Protobuf/Cap'n Proto/Thrift, protocolos binários customizados, filas, brokers, pub/sub, webhooks ou streaming RPC): use [PROTOCOLS-RPC-AND-MESSAGING.md](PROTOCOLS-RPC-AND-MESSAGING.md) (Controles `APP16–APP17`, `PRO03`).
>
> **Alvos de exaustão de recursos e disponibilidade** (trabalho não confiável capaz de consumir CPU, memória, disco, conexões, workers, filas, cotas compartilhadas ou orçamento financeiro do operador): use [RESOURCE-EXHAUSTION-AND-AVAILABILITY.md](RESOURCE-EXHAUSTION-AND-AVAILABILITY.md) (Controles `APP12`, `AI06`, `STK03`).
>
> **Alvos de isolamento e ciclo de vida de dados** (bancos multi-tenant, caches/busca, links de objetos/blobs, analytics, exportação/backup, migração, exclusão/LGPD, retenção ou restauração): use [DATA-ISOLATION-AND-LIFECYCLE.md](DATA-ISOLATION-AND-LIFECYCLE.md) (Controles `APP03`, `DB01–DB08`, `GOV01`).
>
> **Alvos desktop, mobile e IPC local** (apps nativos, deep links, bridges webview, componentes exportados Android/iOS, helpers privilegiados, daemons locais, Unix sockets/XPC/Binder/D-Bus): use [DESKTOP-MOBILE-AND-LOCAL-IPC.md](DESKTOP-MOBILE-AND-LOCAL-IPC.md) (Controle `IPC01`).

**Injection** (subagent_type: `general`)
Rastreie entradas não confiáveis desde o ponto de entrada até sinks perigosos:
- Aplicações web e APIs: consultas SQL/NoSQL, saída HTML, comandos de shell, motores de template, caminhos de arquivos, redirecionamentos HTTP, desserialização.
- Bibliotecas: qualquer função pública que processe dados fornecidos pelo chamador sem validação — operações de buffer, parsers, format strings.
- Ferramentas CLI: construção de comandos de shell, manipulação de caminhos, interpolação de variáveis de ambiente.
- Serviços: construção de queries, serialização de mensagens, log injection, consultas LDAP/XPath.
- Client-side (navegador/JS): DOM XSS, poluição de protótipo, confiança em `postMessage`/origem — cobertos em profundidade pelos blocos de [CLIENT-SIDE.md](CLIENT-SIDE.md) quando selecionados.

Não pare nos caminhos diretos óbvios. Procure **injeção indireta e de segunda ordem**: dados armazenados com segurança inicial e posteriormente recuperados e usados em contexto perigoso por outro módulo. Verifique injeção através de nomes de campos, chaves JSON, cabeçalhos e metadados — não apenas valores — e injeção em sistemas secundários (logs, caches, índices de busca, pipelines analíticos).

**Access control** (subagent_type: `general`)
Verifique se um chamador consegue executar ações fora de sua autoridade legítima. Vá além de checar se existe um `if` de permissão — confirme se ele valida a permissão *correta*, para o recurso *correto*, pelo mecanismo *autoritativo*:
- Existe um caminho alternativo para a mesma mudança de estado que verifica uma permissão mais fraca?
- Um campo no corpo da requisição (*mass assignment* / `tenant_id` no JSON) pode sobrescrever o escopo que o sistema pretendia restringir?
- Há rotas, Server Actions ou resolvers que exigem autenticação, mas esquecem a autorização por objeto (BOLA) ou função (BFLA)?
- O mesmo recurso possui múltiplos caminhos de acesso (REST, GraphQL, webhook, job assíncrono) com checagens inconsistentes?
- Operações em lote (*bulk*, *batch*, exportação, importação) validam permissão individualmente para cada item?

Em modelos de acesso complexos, divida em agentes distintos para bypass de autenticação versus lógica de autorização.

**Resource and file handling** (subagent_type: `general`)
- Path traversal (leitura ou escrita fora dos diretórios pretendidos) — incluindo symlinks, sequências duplamente codificadas e bytes nulos.
- SSRF (fazer o servidor buscar URLs controladas pelo atacante) — incluindo redirecionamentos, DNS rebinding, IPv6 mapeado e diferenciais entre parsers de URL.
- Desserialização insegura, extração de pacotes compactados (*Zip Slip*) e manipulação insegura de arquivos temporários.
- Segurança de memória (se aplicável): buffer overflows, use-after-free, integer overflow.
- Condições de corrida em operações de arquivo (TOCTOU entre checagem e abertura/uso).

**Cryptography and secrets** (subagent_type: `general`)
- Aleatoriedade fraca (`Math.random()`, seeds previsíveis) em valores críticos (tokens, chaves, nonces, códigos OTP).
- Segredos hardcoded, segredos vazados em logs, mensagens de erro, URLs ou bundles enviados ao cliente.
- Derivação fraca de chaves, ausência de verificação HMAC em webhooks, reutilização de nonces em cifras autenticadas.
- Canais laterais de tempo (*timing side-channels*) na comparação de segredos/assinaturas (ausência de `timingSafeEqual`).
- Mau uso de primitivas criptográficas (modo ECB, criptografia sem autenticação/MAC, IVs estáticos, confusão de algoritmos JWT `RS256`/`HS256`).
- Comportamento em caso de falha criptográfica: o caminho de exceção abre em modo inseguro (*fail-open*)?

**Business logic** (subagent_type: `general`)
Caçe erros de lógica de negócio manualmente — scanners automáticos não os enxergam e eles geram achados de altíssimo impacto:
- **Violações de máquina de estados**: É possível pular etapas? Voltar atrás? Reproduzir (*replay*) um fluxo já encerrado? Se o passo 2 de 3 falhar, o passo 1 sofre rollback atômico?
- **Condições de corrida com impacto de negócio**: Operações concorrentes gerando estados inválidos (*double-spend*, aprovação dupla, resgate múltiplo de cupom, *lost updates*) em fluxos *Check-Then-Act* não atômicos.
- **Manipulação numérica e de quantidades**: Valores negativos, zero, overflow, perda de precisão decimal/float, coerção de tipos entre string/array e número.
- **Violações de fronteira funcional**: O input de uma operação legítima contorna uma restrição imposta em outra rota equivalente?
- **Pressupostos implícitos de confiança**: Dados vindos do banco, cache, fila ou plugin tratados como confiáveis porque "foram validados na entrada" — mas gravados por outro caminho sem validação.
- **Lógica temporal**: Expiração, janelas de rate limit, fuso horário e *clock skew* nos milissegundos exatos de fronteira.
- **Comportamento padrão e fallback**: Qual é a postura de segurança quando uma config está ausente, uma *feature flag* está desligada, uma dependência falha ou o sistema está em meio a uma migração?

**Feature abuse and data leakage** (subagent_type: `general`)
Funcionalidades legítimas abusadas para fins não previstos (falhas de design e arquitetura):
- **Exportação/backup como exfiltração**: Um usuário de baixo privilégio consegue disparar exportação, relatório ou snapshot contendo dados acima do seu nível de acesso, de outros tenants, itens excluídos (*soft-deleted*), rascunhos privados ou histórico não expurgado?
- **Importação/restauração como injeção**: A importação sobrescreve registros existentes, contorna validações normais da UI ou injeta objetos em coleções sem permissão de escrita?
- **Busca/filtro/ordenação como oráculo**: Consultas de busca, filtros de atributos ocultos ou ordenação (`ORDER BY` por coluna confidencial) revelam a existência ou o valor de dados inacessíveis pela ordem/contagem dos resultados?
- **Enumeração por efeitos colaterais**: Diferenças em mensagens de erro ("não existe" vs "sem permissão"), tempos de resposta, tamanhos de payload ou códigos HTTP em fluxos de reset, convite ou registro.
- **Vazamento de preview/rascunho/staging**: Tokens de preview escopados frouxamente, rascunhos indexados em buscas/sitemaps/feeds ou cabeçalhos de cache que levam CDNs a servir respostas privadas publicamente.
- **Notificações/webhooks como SSRF**: URLs de callback configuráveis pelo usuário sem validação de rede interna antes e após redirecionamentos.

**Chained vulnerabilities and trust boundaries** (subagent_type: `general`)
Comportamentos individualmente contidos que se tornam vulnerabilidades graves quando encadeados entre componentes:
- **Falhas de fronteira em múltiplos passos**: Mapeie o que um principal de baixo privilégio pode ler, gravar, invocar e reter, conectando apenas saídas concretas comprovadas a decisões de confiança posteriores.
- **Lacunas de confiança entre componentes**: O Componente A valida a entrada e a repassa ao Componente B — compare exatamente o que A garante com o que B pressupõe (truncamento, coerção de tipo, normalização Unicode/path, escopo de tenant).
- **Uso de segunda ordem**: Um nome de campo vira JSON path, um slug vira caminho de arquivo, texto escapado entra em renderizador raw, ou string armazenada vira URL, regex, template ou expressão de política.
- **Crescimento de escopo e capacidade**: Tokens, API keys, sessões OAuth, permissões MCP ou capacidades de agentes que se ampliam indevidamente após delegação, refresh, cache, mudança de papel ou composição.
- **Janelas de tempo e ordenação**: Janelas entre setup, migração, soft-delete, revogação/expiração de cache e *validate-then-consume*.
- **Rollback e recuperação**: Undelete, restauração de revisão e cancelamento devem reaplicar propriedade, validação e autorização vigentes.

**Wildcard** (subagent_type: `general`)
Você não recebe uma categoria fixa: encontre vulnerabilidades fora das classes padrão já atribuídas.

Leia o código que parece entediante ou desconectado de segurança, mantendo o mesmo rigor de fronteira e validação das demais classes:
- Qual é o trecho mais estranho da base de código? Por que ele existe e o que acontece se for abusado?
- Existem funcionalidades pela metade, experimentais, legadas ou acopladas às pressas? Elas costumam ter a segurança mais frágil por receberem menos revisão.
- O que acontece ao chamar a API de maneiras que o frontend jamais faria?
- Existem rotas, parâmetros, cabeçalhos ou modos ocultos não documentados nos middlewares e roteadores?
- O que ocorre ao combinar recursos que não foram desenhados juntos (ex.: localização + preview + cache; importação + plugins + webhooks; OAuth + impersonation + API keys)?
- Há pistas críticas no histórico Git (correções revertidas, checagens comentadas, segredos antigos)?
- Quais ações irreversíveis deixam de vincular aprovação ao principal, ação e recurso finais?
- Quais pressupostos ambientais frágeis o código assume (relógio exato, DNS confiável, filesystem case-sensitive)?
- Olhe para os arquivos de teste: o que eles **não** estão testando?

Siga anomalias dentro do seu escopo até decidir o invariante. Se um comentário explicar "por que isto é seguro", verifique se a explicação é realmente verdadeira no código atual.

**Obvious things** (subagent_type: `general`)
Enquanto outros agentes caçam falhas sutis, este agente verifica exposições fundamentais frequentemente esquecidas porque todos presumem que alguém já conferiu:
- Senhas, API keys, tokens ou chaves privadas hardcoded no código (`password`, `secret`, `apikey`, `token`, `Bearer`, `-----BEGIN`).
- Comentários `TODO`/`FIXME`/`HACK`/`XXX` ligados a segurança (`TODO: add auth`, `HACK: skip permission check`).
- Modo debug/dev ativável em produção por variável de ambiente, query param ou cabeçalho.
- Credenciais de seed/teste/exemplo ativas em caminhos produtivos.
- Endpoints `/debug`, `/admin`, `/test`, `/status`, `/health`, `/metrics`, `/env`, `/.env`, `/config` desprotegidos.
- Arquivos `.env`, `credentials.json`, `*.pem`, `*.key` versionados ou `.gitignore` incompleto.
- Dependências sem lockfile/pinning ou com CVEs críticas alcançáveis.
- Uso de `eval()`, `exec()`, `child_process`, `Function()`, `vm.runInContext` ou `import()` com entrada dinâmica.
- Cabeçalhos CORS excessivamente permissivos com credenciais.
- Cookies sensíveis sem `HttpOnly`, `Secure` ou `SameSite`.
- Open redirects (`redirect`, `return`, `next`, `url`, `goto`, `continue` sem validação estrita).
- Endpoints sem TLS ou respostas de erro em produção vazando stack traces, caminhos internos ou erros SQL.

**Importante**: Para qualquer item encontrado aqui, comprove o caminho completo no código — não apenas a aparência superficial. Um flag ausente isoladamente não é um achado confirmado sem demonstrar o dado sensível ou a fronteira efetivamente impactada.
