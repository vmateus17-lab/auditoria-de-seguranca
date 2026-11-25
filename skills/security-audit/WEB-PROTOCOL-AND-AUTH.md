# Caça em Protocolos HTTP e Autenticação (`HTTP-Protocol and Authentication Hunting`)

#### When to use this file

Utilize este guia quando o alvo operar sobre HTTP em uma fronteira de parsing, cache, autenticação de navegador ou identidade federada: aplicações web, APIs, proxies reversos, CDNs, gateways, servidores HTTP customizados e serviços que implementam sessões, JWT/PASETO, OAuth 2.1/OIDC, SAML, recuperação de senha, MFA, passkeys/WebAuthn, API keys ou mTLS. Enquanto a revisão de controle de acesso em `ATTACK-CLASSES.md` pergunta se um principal pode executar uma operação, este guia investiga se a camada HTTP ou de identidade pode confundir *a qual* principal, requisição, nível de garantia (*assurance*) ou token aquela operação pertence (veja também controles `APP01–APP02`, `APP08`, `APP18`, `APP20`, `PRO01–PRO02`).

## Core discipline (incluir em todo prompt de agente para este domínio)

```
- Achados de framing HTTP e cache exigem duas interpretações divergentes da mesma requisição, resposta ou chave. Nomeie ambos os componentes e o valor exato normalizado em cada lado.
- Para cada credencial ou token, localize a verificação criptográfica e todos os vínculos obrigatórios ao seu papel: issuer (iss), audience (aud), origin, RP ID, client_id, sessão, principal, recurso, nível de autenticação (assurance), expiração e estado de uso único.
- Host, Forwarded, X-Forwarded-*, Origin, Referer, alvos de redirect, state de callback e URLs derivadas da requisição são decisões de confiança. Rastreie cada um até a identidade ou resposta afetada.
- Um cabeçalho ausente, atributo de cookie faltante, tela de MFA omitida ou ausência de rate limit isoladamente não constitui achado confirmado: exija aceitação de requisição inválida, impacto cross-principal, downgrade de garantia ou vazamento real de credencial.
- Classifique como `confirmed` apenas com evidência completa no código e testes locais delimitados de requisição/token. Use `needs_validation` quando proxy gerenciado, IdP externo, navegador, certificado ou configuração implantada for decisiva mas invisível no repositório.
```

## HTTP framing and cache attack classes (subagent_type: `general`)

**Request framing and desynchronization**
Frontend (proxy/CDN) e backend discordam sobre tamanho da requisição ou normalização de cabeçalhos (*HTTP Request Smuggling*). Revise múltiplos `Content-Length`, `Transfer-Encoding`, downgrade HTTP/2 ou HTTP/3 para HTTP/1.1, normalização de nomes de headers, cabeçalhos `Connection` proibidos e conversões CR/LF. Confirme quais bytes cada par atribui a cada requisição.

**Web cache poisoning through unkeyed input**
Um valor da requisição altera o conteúdo armazenado em cache ou cabeçalhos sensíveis, mas não faz parte da *cache key* (ex.: `X-Forwarded-Host`, `X-Forwarded-Scheme`, cookies selecionados, headers de idioma/dispositivo ou normalização de query).

**Cache deception and private-response caching**
O roteamento de borda trata um caminho dinâmico autenticado como ativo estático público (ex.: `/api/perfil/dados.css`), ou armazena respostas privadas sem considerar estado de autorização e diretivas `Cache-Control: private, no-store`.

**Host and forwarded-header trust**
Metadados de host/proxy não confiáveis determinam URLs absolutas de reset de senha, roteamento de tenant, callbacks OAuth, chaves de cache ou IP do cliente usado em políticas de autorização/rate-limit sem que o ingress confiável remova cópias enviadas pelo cliente.

**Response-header injection**
Dados não confiáveis chegam a `Location`, `Set-Cookie`, `Content-Security-Policy` ou outros headers de resposta contendo CRLF ou caracteres de controle não rejeitados pelo runtime.

## Browser-session attack classes (subagent_type: `general`)

**Ordinary CSRF**
O navegador envia credenciais ambientes (cookies) a um endpoint mutável que aceita requisições cross-site sem token anti-CSRF efetivo, vínculo SameSite compatível ou validação estrita de `Origin`/`Referer`. Inventarie mutações via formulário, JSON, multipart, *method-override* e rotas legadas, incluindo *login CSRF*.

**Session fixation and invalidation**
Identificadores de sessão não rotacionados após login, troca de conta, conclusão de MFA ou impersonation, ou tokens/sessões/conexões WebSocket que continuam válidos após logout, alteração de senha, revogação ou desativação da conta.

**Cookie scope and transport**
Cookies sensíveis com `Domain` amplo demais, `Path` permissivo, transporte sem `Secure` ou colisão (*cookie tossing*) com cookies irmãos de subdomínios menos confiáveis.

## Federated-identity attack classes (subagent_type: `general`)

Estabeleça primeiro o papel do componente (Authorization Server vs Relying Party/Client).

**JWT verification and claim binding**
Verifique validação de assinatura, algoritmo fixado no servidor (proibindo `none` e confusão `RS256`→`HS256`), origem da chave (`kid`, `jku`, `x5u` tratados como seletores não confiáveis), checagem de `exp`, `nbf`, `aud` e `iss`, e ausência de caminhos que usam `jwt.decode()` sem `verify()`. Um token válido emitido para outro serviço é inválido aqui mesmo assinado pelo mesmo IdP.

**OAuth/OIDC request and callback binding**
Valide correspondência exata de `redirect_uri`, `state` vinculado à sessão, PKCE (`S256`) vinculado ao authorization code, `nonce`/assinatura/audiência do ID token e vinculação ao IdP selecionado em fluxos multi-provedor.

**SAML signed-object and assertion binding**
Garanta que o elemento XML cuja assinatura criptográfica foi verificada é exatamente o mesmo elemento consumido como identidade (*XML Signature Wrapping*), com parser seguro contra XXE, checagem de audiência/destinatário e proteção contra replay.

## MFA, passkey, and account-transition attack classes (subagent_type: `general`)

**MFA enrollment and assurance downgrade**
Cadastro, substituição, desativação de MFA, geração de códigos de recuperação e criação de dispositivos confiáveis devem exigir autenticação forte recente (*fresh step-up*), impedindo que o primeiro fator sozinho substitua o segundo fator.

**Step-up binding and bypass**
Um desafio MFA concluído eleva a sessão, conta, tenant ou operação errada, ou uma rota paralela de API/batch executa a mesma operação sensível omitindo a exigência de step-up.

**WebAuthn and passkey verification**
No registro, vincule `challenge`, `rpId`, `origin` esperada, credencial, `userHandle`, algoritmo e verificação de usuário exigida à sessão iniciadora. Na autenticação, valide `challenge` de uso único, `rpId`/`origin`, pertencimento da credencial à conta, assinatura e flags de presença/verificação de usuário, prevenindo confusão de `userHandle` em fluxos *discoverable*.

**Account linking and identity collision**
Vincular um IdP social, passkey, e-mail, telefone ou carteira externa a uma conta existente exige sessão autenticada atual, prova verificada de posse da nova identidade, step-up aplicável e `state` de callback estritamente preso à conta iniciadora (evitando *pre-hijacking* ou colisão de identificadores não verificados).

**Password reset and broader recovery**
Tokens de reset, recuperação via suporte, códigos de backup e troca de e-mail/telefone costumam ser o elo mais fraco da autenticação: valide entropia criptográfica, vínculo usuário/ação, expiração curta, uso único atômico, invalidação de tokens/sessões anteriores e confiança na URL de entrega.

## API-key and mTLS attack classes (subagent_type: `general`)

**API-key scope and resource binding**
Chaves que autenticam em tenants, ambientes, recursos ou métodos além do escopo gravado no servidor, confusão entre chaves públicas (*publishable*) e secretas, ou caches de revogação que mantêm chaves canceladas ativas.

**API-key exposure and unsafe transport**
Chaves secretas expostas em bundles frontend, query strings, redirecionamentos, logs, artefatos de build ou respostas acessíveis a principais de menor privilégio.

**mTLS peer and application-identity confusion**
Aplicação que confia em cabeçalhos `X-SSL-Client-*` vindos de qualquer par de rede sem restringir ao proxy terminador confiável, ou que valida a cadeia X.509 mas mapeia campos `Subject`/`SAN` influenciáveis de forma ambígua para contas internas.

**Certificate lifecycle fallback**
Certificados expirados, revogados ou ausentes que provocam fallback silencioso (*fail-open*) para modo anônimo ou apenas bearer, ou conexões persistentes em pool que retêm autorização após revogação.

## Universal moves (aplicar transversalmente neste domínio)

- Percorra o ciclo completo `emitir → armazenar → transmitir → consumir → renovar → revogar` para cada credencial e desafio, comparando fluxos normais, erros, retentativas, migração, rotas legadas e troca de conta.
- Enumere todas as portas para a mesma identidade e todas as rotas para a mesma operação sensível: a política efetiva do sistema é sempre a do caminho paralelo mais fraco.
- Compare lado a lado a normalização entre parser HTTP, proxy, roteador, cache e aplicação usando fixtures locais idênticas em sandbox.
- Em recuperação e vinculação de contas, desenhe o grafo antes/depois da conta verificando prova de posse, nível de garantia e efeito de revogação em cada aresta.

## Validation rules (aplicar antes de reportar QUALQUER achado aqui)

1. Aplique rigorosamente o filtro de visibilidade do código-fonte: cadeias de proxy, chaves de cache na CDN, políticas de IdP externo, cadeia de certificados e flags de deploy fora do repositório exigem `needs_validation`.
2. Em achados de framing HTTP e cache, nomeie os dois componentes e o parse/chave divergente, comprovando impacto cross-request, cross-user ou exposição de resposta privada com fixtures locais delimitadas.
3. Em achados de JWT, OAuth, SAML, MFA, passkey, vinculação de conta, recuperação, API key e mTLS, cite a linha exata da verificação e o vínculo ausente, demonstrando que o servidor aceita a transição ou credencial inválida.
4. Em CSRF, nomeie a credencial ambiente, a rota mutável, o formato aceito de requisição cross-site, a política de cookie e a checagem ausente (rotas somente-leitura ou protegidas por header `Authorization: Bearer` não ambiente não se qualificam).
5. Verifique os padrões nativos da versão exata do framework antes de reportar; se desconhecidos, use `needs_validation`.
6. Retorne `confirmed` somente com trace completo e ganho observável de identidade, mutação de estado ou vazamento; para `needs_validation`, especifique o fato ausente e o check seguro correspondente.
