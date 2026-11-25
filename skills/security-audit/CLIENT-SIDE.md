# Caça Client-Side e Navegador (`Client-Side and Browser Hunting`)

#### When to use this file

Utilize este guia quando decisões relevantes de confiança ou renderização de dados não confiáveis ocorrerem no navegador: Single-Page Applications (SPAs), aplicações SSR/hidratadas (React/Next.js/Vue), extensões de navegador, webviews embarcadas, Service Workers, aplicações offline/PWA e código que manipula o DOM, recebe mensagens entre janelas (`postMessage`), utiliza WebSockets ou persiste dados no armazenamento do navegador. Inclui fontes que o servidor jamais enxerga, como fragmentos de URL (`location.hash`), `window.name`, `postMessage` e caches locais (veja também controles `APP07–APP08`, `STK02` e `CLI01`).

Use em conjunto com `ATTACK-CLASSES.md`. Para o lado nativo de bridges WebView, combine com `DESKTOP-MOBILE-AND-LOCAL-IPC.md`; para sessões e callbacks de autenticação no servidor, combine com `WEB-PROTOCOL-AND-AUTH.md`.

## Core discipline (incluir em todo prompt de agente para este domínio)

```
- Todo candidato client-side exige uma fonte controlável (source) e um sink de execução ou vazamento. Nomeie ambos e demonstre os dados influenciados pelo atacante alcançando o sink.
- O impacto deve atingir a sessão de uma vítima, outra origem ou persistência compartilhada. Autoinjeção (self-XSS) e exibição dos próprios dados do atacante não são vulnerabilidades.
- Escapamento automático de frameworks, Same-Origin Policy (SOP), CSP, COOP/CORP, escopo de Service Worker e `noopener` padrão dos navegadores modernos são controles reais: verifique-os antes de atribuir impacto.
- Armazenamentos e caches do navegador são compartilhados por origem e podem sobreviver ao logout. Identifique quem grava, quem lê e qual ciclo de vida de conta, tenant ou worker limpa cada registro.
- Use `confirmed` apenas com evidência completa no código somada a testes locais delimitados de navegador/DOM. Use `needs_validation` quando headers de borda, permissões de extensão ou comportamento de renderizador externo forem decisivos e não estiverem visíveis no repositório.
```

## DOM and object-state attack classes (subagent_type: `general`)

**DOM-based XSS**
Rastreie propriedades de `location`, `document.referrer`, `window.name`, payloads de `postMessage`, storage e estado controlado pelo navegador até `innerHTML`, `outerHTML`, `document.write`, `dangerouslySetInnerHTML`, `v-html`, avaliadores dinâmicos de código, URLs executáveis (`javascript:`, `data:`) ou escape hatches de bibliotecas UI. Interpolação padrão escapada pelo framework não é vulnerabilidade.

**DOM clobbering**
Atributos `id` ou `name` injetados em HTML sanitizado parcialmente sombreiam variáveis globais, propriedades de `document`/`window`, objetos de configuração ou flags de inicialização consumidos posteriormente por scripts confiáveis. Exija tanto a preservação do atributo no markup quanto o uso sensível à segurança do valor sombreado.

**Prototype pollution and gadget chain**
Uma chave controlada pelo atacante (`__proto__`, `constructor.prototype`) alcança uma escrita recursiva (deep merge, clone, atribuição por path) alterando `Object.prototype`, seguida por um *gadget* alcançável que consome a propriedade poluída para alterar autorização, execução, navegação ou renderização. `JSON.parse` puro, cópia rasa ou poluição sem gadget alcançável não bastam para confirmação.

## Cross-origin messaging and network attack classes (subagent_type: `general`)

**`postMessage` origin and source trust**
Um listener executa ação sensível ou mutação de estado com `event.data` sem validar uma allowlist exata de `event.origin` (rejeitando verificações frágeis por `indexOf`, `startsWith`, `endsWith` ou regex sem âncoras) e, quando múltiplos frames compartilham origem, `event.source`. No envio, verifique se dados confidenciais são transmitidos com `targetOrigin: "*"`.

**Cross-site WebSocket request use**
Um upgrade WebSocket aceita cookies ambientes de uma origem externa sem validação estrita do cabeçalho `Origin` nem token específico de canal, permitindo que uma página maliciosa leia ou altere estado na sessão da vítima (*CSWSH*).

**Credentialed CORS trust**
O servidor reflete ou valida fracamente o cabeçalho `Origin` em conjunto com `Access-Control-Allow-Credentials: true` em rotas que retornam dados privados ou aceitam ações sensíveis. (Nota: `Access-Control-Allow-Origin: *` com `Credentials: true` é bloqueado nativamente pelos navegadores; reporte apenas origens efetivamente refletidas/aceitas).

## Service-worker and browser-storage attack classes (subagent_type: `general`)

**Service-worker registration and scope takeover**
Conteúdo influenciável pelo atacante pode controlar a URL do script registrado no Service Worker, abusar de `Service-Worker-Allowed` excessivamente amplo ou alterar `importScripts` sem controle de integridade.

**Service-worker cache and identity confusion**
O Service Worker armazena respostas personalizadas em Cache Storage sem incluir conta, tenant ou estado de autorização na chave/política e sem expurgar no logout ou troca de conta, servindo dados privados do usuário anterior em navegações subsequentes ou modo offline.

**Browser-storage disclosure and stale authorization**
Tokens, respostas confidenciais, dados serializados de hidratação (`__NEXT_DATA__`) ou decisões de permissão persistem em `localStorage`, `sessionStorage`, IndexedDB ou Cache Storage e permanecem legíveis por outra conta no mesmo navegador compartilhado ou continuam autorizando ações após logout/revogação.

**Cross-context storage and broadcast confusion**
Eventos `storage`, `BroadcastChannel`, `SharedWorker` ou caches de origem propagam identidade ou comandos privilegiados entre abas sem vinculá-los à sessão ativa atual, causando confusão em trocas de conta ou tenant.

## Cross-site information leak classes (subagent_type: `general`)

**XS-Leaks and cross-origin state oracles**
Uma página externa distingue estado confidencial cross-origin através de eventos `onload`/`onerror`, contagem de frames, redirecionamentos, status/tamanho de resposta, cache probing ou timing enquanto o navegador anexa credenciais da vítima. Exija um predicado confidencial concreto comprovado (ex.: existência de objeto privado ou papel privilegiado).

**Window and opener state disclosure**
Relacionamentos `window.opener` retidos explicitamente ou janelas nomeadas permitem que páginas externas influenciem navegações privilegiadas ou infiram estado protegido onde `Cross-Origin-Opener-Policy` (COOP) e `noopener` não isolam o contexto.

## UI-redress and navigation attack classes (subagent_type: `general`)

**Clickjacking**
Uma ação sensível de mudança de estado pode ser enquadrada em `<iframe>` externo por ausência efetiva de `Content-Security-Policy: frame-ancestors` ou `X-Frame-Options` e ausência de isolamento de interação. Ausência de cabeçalhos em páginas puramente públicas/somente-leitura é nota de hardening.

**Client-side navigation confusion**
Fontes controladas pelo cliente definem destinos de `window.location`, `router.push` ou `window.open` sem restringir esquemas (`javascript:`, `data:`) ou domínios confiáveis.

## Universal moves (aplicar transversalmente neste domínio)

- Comece pelos sinks de DOM, navegação, Service Worker, `postMessage` e storage, rastreando de trás para frente até fontes exclusivas do navegador e fontes vindas do servidor/IA.
- Teste troca de conta, logout, atualização de worker, fallback offline e abas antigas usando origem local isolada e contas fictícias.
- Para XS-Leaks, documente apenas predicados comprovados pelo código e comportamento local do navegador, apontando o cabeçalho (`COOP`, `CORP`, `SameSite`, `frame-ancestors`) que elimina o oráculo.

## Validation rules (aplicar antes de reportar QUALQUER achado aqui)

1. Cite exatamente a fonte (`source`), o `sink`, a política do navegador aplicável, a origem/sessão afetada e a mutação ou vazamento observável.
2. Em poluição de protótipo, comprove tanto a escrita recursiva quanto o gadget relevante. Em DOM clobbering, comprove que a marcação sobrevive ao sanitizador e que o valor sombreado é consumido.
3. Em Service Workers e armazenamento local, comprove alcançabilidade de ciclo de vida: o dado ou cache precisa alcançar outra conta, outro tenant ou sobreviver indevidamente após revogação/logout.
4. Em `postMessage`, CORS, WebSocket e XS-Leaks, demonstre a falha na validação de origem e confirme que CSP, COOP/CORP e políticas de cookies não bloqueiam o vetor.
5. Retorne `confirmed` somente com caminho client-side completo e prova local delimitada; retorne `needs_validation` especificando o cabeçalho de borda, permissão de extensão ou política de deploy que o proprietário deve verificar.
