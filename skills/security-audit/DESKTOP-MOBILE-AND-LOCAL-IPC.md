# Caça em Desktop, Mobile e IPC Local (`Desktop, Mobile, and Local IPC Hunting`)

#### When to use this file

Utilize este guia quando o alvo for uma aplicação desktop (Electron, Tauri, macOS/Windows/Linux nativa) ou mobile (Android, iOS, React Native, Flutter), helper privilegiado, atualizador local, daemon de sistema, host de WebView, handler de deep links/custom schemes, host de *native messaging* de navegador ou cliente/servidor de IPC local (Unix domain sockets, Named Pipes, XPC, Android Binder, D-Bus, loopback local). Atores relevantes de menor confiança podem ser documentos abertos pelo usuário, conteúdo web remoto, outro app instalado, outro usuário do SO ou um processo filho em sandbox (veja também controle `IPC01`).

Combine com `CLIENT-SIDE.md` para o lado web da WebView, `MEMORY-SAFETY-AND-BINARY.md` para corrupção de memória nativa e `SUPPLY-CHAIN-AND-RELEASE.md` para autenticidade de atualizações.

## Core discipline (incluir em todo prompt de agente para este domínio)

```
- Estabeleça claramente a capacidade inicial realista do atacante: outro app local sem privilégio, outro usuário do SO, processo filho em sandbox, arquivo baixado ou origem web remota. Auto-sabotagem pelo próprio usuário com a mesma autoridade de SO não constitui violação de fronteira.
- Caminhos de executáveis, nomes de processos, bundle/package IDs autodeclarados e campos no payload JSON não são autenticação de par (peer authentication): exija credenciais de par impostas pelo kernel do SO (SO_PEERCRED/LOCAL_PEERPID, assinatura de código/audit token XPC, permissões UID/GID de socket ou tokens criptográficos de canal).
- Toda bridge nativa ou helper privilegiado deve autorizar cada operação e recurso final após o parsing completo. Uma UI confiável não torna confiáveis argumentos influenciáveis por conteúdo externo.
- Sandboxes do SO, assinaturas de código, entitlements, permissões de manifesto, ACLs de Keychain/Keystore e políticas de componentes exportados são controles reais quando visíveis no repositório.
- Use `confirmed` com evidência completa no código e testes locais/emulador delimitados. Use `needs_validation` quando assinatura final de release, merge externo de manifestos, versão do SO ou ACLs do instalador forem decisivas mas invisíveis no código.
```

## Deep-link, callback, and navigation attack classes (subagent_type: `general`)

**Custom-scheme and deep-link ambiguity**
Outro aplicativo ou página web aciona um URI customizado (`meuapp://...`) ou Universal/App Link que altera estado, importa configurações, conclui login ou abre arquivos locais sem vínculo criptográfico à sessão atual e sem validação rigorosa de host/path/parâmetros duplicados.

**App and account handoff confusion**
Callbacks OAuth, SSO, Magic Link ou pagamentos no mobile/desktop que podem ser interceptados por outro app que registrou o mesmo custom scheme (ausência de App Links / Universal Links verificados com PKCE + `state` vinculado à sessão iniciadora).

**File-open and intent authority confusion**
Abertura de arquivos associados, Intents de compartilhamento Android (`content://` / `file://`), itens de drag-and-drop ou ações de notificação que disparam execução privilegiada ou cópia arbitrária de arquivos privados do app (*file provider* / symlink / path traversal).

## Webview and native-bridge attack classes (subagent_type: `general`)

**Navigation-origin to bridge confusion**
Frames remotos, redirecionamentos ou páginas de erro/fallback carregadas em WebView/Electron alcançam uma bridge nativa (`addJavascriptInterface`, `WKScriptMessageHandler`, `ipcRenderer`/`contextBridge` com `nodeIntegration` inseguro) destinada apenas a páginas empacotadas confiáveis. Valide a origem exata no momento de *cada* chamada da bridge e restrinja navegações/subframes.

**Over-broad native bridge capabilities**
Conteúdo web capaz de invocar métodos genéricos na bridge para ler/gravar arquivos arbitrários, abrir processos do sistema (`shell.openExternal` sem allowlist de esquemas `https:`), ler credenciais ou enviar mensagens IPC irrestritas.

**Webview file and universal access**
Configurações inseguras (`allowFileAccessFromFileURLs`, `allowUniversalAccessFromFileURLs`, `webSecurity: false`, handlers de esquemas customizados sem isolamento de origem) que permitem a leitura de arquivos privados do app ou cross-origin no dispositivo.

## Local IPC and exported-component attack classes (subagent_type: `general`)

**IPC peer-authentication gaps**
Unix sockets com permissões `0666`/`0777`, Named Pipes sem DACL restrita, serviços XPC sem verificação de *audit token*/code signing requirement, serviços D-Bus/Binder ou servidores HTTP em `127.0.0.1` (vulneráveis a requisições do navegador via DNS rebinding ou CORS/CSRF local e a outros processos locais) que aceitam comandos sensíveis sem autenticar o par.

**Claimed principal versus channel identity**
O canal IPC autentica um processo/usuário, mas o handler confia em um `uid`, `profile_id` ou caminho informado dentro da mensagem em vez de derivá-lo da credencial de kernel do par conectado.

**Exported service, activity, receiver, or provider overreach**
Activities, Services, BroadcastReceivers ou ContentProviders Android (`android:exported="true"` ou implícito via `intent-filter`) invocáveis por apps terceiros sem permissão `signature`, expondo dados internos em SQLite/arquivos via `grantUriPermissions` ou executando ações internas.

**IPC lifecycle and correlation confusion**
IDs de requisição previsíveis, descritores herdados por processos filhos ou diretórios de socket graváveis por terceiros permitindo sequestro ou substituição do canal IPC.

## Privileged-helper and local-file attack classes (subagent_type: `general`)

**Privileged helper as confused deputy**
Um daemon `root`/`SYSTEM` ou *privileged helper tool* aceita parâmetros de linha de comando, caminhos de scripts ou configurações passados pelo cliente sem privilégio, permitindo escalonamento local de privilégio (LPE).

**Install, update, and repair path trust**
Instaladores ou atualizadores rodando com privilégio elevado que executam binários, scripts ou descompactam arquivos em diretórios graváveis pelo usuário comum após a checagem inicial.

**Local file ownership and TOCTOU**
Um processo privilegiado verifica caminho/permissão por string (`stat`) e em seguida abre/escreve (`open`/`chmod`/`chown`) seguindo symlinks, hard links ou junções NTFS criados na janela de corrida. Exija operações relativas a descritor com `O_NOFOLLOW`.

**Credential-store and local-secret boundary mismatch**
Tokens ou chaves armazenados fora do Keychain/Keystore seguro com permissões legíveis por outros usuários do SO, incluídos em backups não criptografados do dispositivo (`allowBackup="true"`) ou vazados na área de transferência/notificações na tela bloqueada.

## Application-state and device-lifecycle attack classes (subagent_type: `general`)

**Account switch, logout, and device restore leakage**
Bancos SQLite locais, caches de WebView, tarefas em background, widgets ou aprovações biométricas que sobrevivem ao logout ou troca de conta e expõem dados ao próximo usuário do aplicativo.

**Pending-action and user-presence confusion**
Prompts biométricos ou notificações interativas que apenas liberam um booleano em UI sem vincular criptograficamente a chave no Secure Enclave/Keystore (`CryptoObject`) ou sem prender a aprovação à operação normalizada exibida na tela.

## Universal moves (aplicar transversalmente neste domínio)

- Enumere cada processo, componente mobile, porta/socket local, URI scheme, associação de arquivo, origem de WebView e helper privilegiado, anotando usuário de SO, privilégio, quem pode chamá-lo e qual operação ele executa.
- Inspecione os artefatos finais de empacotamento: `AndroidManifest.xml` mesclado, `Info.plist`, entitlements macOS/iOS, `electron-builder` configs e scripts de instalação.
- Valide apenas em ambiente/emulador isolado com perfis e arquivos fictícios.

## Validation rules (aplicar antes de reportar QUALQUER achado aqui)

1. Nomeie a capacidade inicial do atacante, a fronteira de SO/app cruzada, o canal de entrada, o argumento aceito e a operação privilegiada ou vazamento resultante.
2. Confirme os controles de sandbox do SO, credenciais de par, assinatura de código, entitlements e manifesto visíveis no código; fatos pós-empacotamento ausentes exigem `needs_validation`.
3. Em bridges WebView, cite tanto o caminho de navegação/origem quanto o sink nativo privilegiado. Em IPC/helpers, cite a falha na autenticação do par e no destino normalizado final.
4. Pare imediatamente após demonstrar o resultado mínimo de fronteira com dados fictícios (nunca instale persistência real no host).
5. Retorne `confirmed` somente com cadeia completa no código e prova local; retorne `needs_validation` indicando o atributo de manifesto final, assinatura, versão de SO ou ACL de instalador a verificar.
