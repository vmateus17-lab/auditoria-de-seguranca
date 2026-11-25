# Caça em Segurança de Memória, Binários e Kernel (`Memory Safety, Binary, and Kernel Hunting`)

#### When to use this file

Utilize este guia quando o alvo processar bytes não confiáveis em contexto sem segurança automática de memória ou altamente privilegiado: C, C++, Objective-C, blocos `unsafe` em Rust, extensões nativas Node-API/Python C-API/CGo/JNI (FFI), módulos e drivers de kernel, parsers/decoders de mídia/arquivos/rede, daemons nativos, firmware, loaders binários, runtimes de linguagens e compiladores JIT (veja também controle `MEM01`). Combine com `PROTOCOLS-RPC-AND-MESSAGING.md` para lógica de autorização de protocolo.

## Core discipline (incluir em todo prompt de agente para este domínio)

```
- Rederive cada limite de tamanho (bound) e tempo de vida (lifetime) a partir das entradas controladas pelo atacante e de todos os chamadores, validando contra o pior caso aceito e não contra vetores típicos.
- Um panic, alerta de sanitizer ou crash só comprova vulnerabilidade de segurança quando uma entrada não confiável realista o alcança através de uma fronteira de confiança. Não extrapole automaticamente qualquer crash para corrupção de memória explorável (RCE) ou negação de serviço compartilhada sem evidência.
- Valide em harness local isolado com sanitizers (ASan, UBSan, MSan, TSan), testes determinísticos de concorrência e alvos existentes de fuzzing. Pare imediatamente ao comprovar o invariante violado e o efeito observável; não desenvolva exploits pós-corrupção.
- Código Assembly inline, JIT, alocadores customizados, acessos intra-struct e bibliotecas FFI externas podem escapar da instrumentação dos sanitizers: identifique exatamente quais instruções estão cobertas.
- Classifique como `confirmed` apenas após evidência completa no código e validação local delimitada comprovarem defeito e efeito. Use `needs_validation` quando fatos de ABI externa, alocador, arquitetura de hardware ou alcançabilidade em deploy permanecerem desconhecidos.
```

## Bounds, integer, and representation attack classes (subagent_type: `general`)

**Out-of-bounds read or write**
Comprimentos, offsets, índices ou terminadores nulos alcançam buffers estáticos ou alocados no heap sem verificação exata de limite. Recalcule o espaço disponível considerando cabeçalhos, alinhamento, padding e terminador `\0`, verificando tanto a capacidade de origem quanto a de destino e leituras anteriores à validação do comprimento declarado.

**Integer overflow, underflow, truncation, and signedness**
Revise toda aritmética controlada pela entrada antes de chamadas `malloc`/`realloc`, `memcpy`, loops, indexação e aritmética de ponteiros. Padrões clássicos de alto risco: `a - b` com `b > a` (underflow unsigned), `count * sizeof(T)` (overflow na alocação gerando heap buffer overflow na cópia), somas próximas a `INT_MAX`/`SIZE_MAX`, conversão implícita de números negativos `int` para `size_t` e truncamento de comprimentos de 64 bits para campos de 32/16 bits.

**Unit and pointer-depth confusion**
Mistura acidental entre bytes, número de elementos, code units UTF-16/UTF-32, palavras de máquina ou tamanhos de ponteiros aninhados (ex.: alocar `n` bytes mas copiar `n * sizeof(uint32_t)`, ou somar offset já multiplicado a um ponteiro tipado `T*`).

**Uninitialized or partially initialized data**
Buffers na stack/heap, bytes de padding em structs C/Rust (`repr(C)`), campos opcionais ou capacidade extra de vetores (`Vec::set_len` inseguro) serializados, hasheados ou retornados através de uma fronteira de confiança antes de inicialização completa (*infoleak* de memória).

## Lifetime, type, and concurrency attack classes (subagent_type: `general`)

**Use-after-free, stale view, and double free**
Objetos liberados enquanto callbacks assíncronos, filas de espera, timers, iteradores, *slices* emprestados (`&[u8]` derivado de ponteiro cru) ou ponteiros cacheados ainda podem acessá-los. Inspecione todos os caminhos de erro precoce, cancelamento, fechamento e `realloc` (que invalida ponteiros anteriores ao mover o bloco).

**Type confusion and invalid downcast**
Tags de união discriminada (`union`), vtables C++, handles opacos ou enums desserializados verificados de um modo na validação e lidos sob outro tipo no consumo (`static_cast` sem checagem dinâmica ou tag obsoleta após reutilização de slot).

**Reference-count and ownership races**
Incremento/decremento não atômico de contadores de referência entre threads, checagem seguida de uso fora do mutex ou regras inconsistentes de posse entre caminho rápido (*fast path*) e caminho de erro/shutdown.

**Shared-state races and TOCTOU**
Streams concorrentes, caches globais, inicialização preguiçosa (*lazy init*) sem primitiva `Once` e destruição concorrente de recursos invalidando limites ou ponteiros já verificados. Comprove com escalonamento local repetível ou ThreadSanitizer (TSan).

**Lock-order, deadlock, and starvation**
Operações alcançáveis externamente adquirindo múltiplos locks em ordem invertida ou mantendo mutexes retidos durante I/O bloqueante de rede.

## FFI and ABI attack classes (subagent_type: `general`)

**Pointer-length and ownership contract mismatch**
Código gerenciado (Rust seguro, Go, Node.js, Python, Java) e código nativo (`extern "C"`) discordando sobre quem aloca, quem libera, se o buffer pode ser movido pelo Garbage Collector sem *pinning*, por quanto tempo o ponteiro permanece válido em chamadas assíncronas ou se o tamanho indica bytes vs caracteres.

**Layout, alignment, and enum disagreement**
Passagem de structs, bitfields, registros `#pragma pack`, assinaturas de callbacks ou enums através de FFI com divergência de alinhamento (`#[repr(C)]` ausente), largura de inteiros (`c_long` 32 vs 64 bits Windows/Linux) ou convenção de chamada.

**Unwind, exception, and thread-affinity violations**
Exceções C++ ou `panic!` do Rust atravessando fronteiras `extern "C"` sem captura (comportamento indefinido / abort imediato do processo compartilhado) ou invocação de APIs presas a uma thread específica a partir de um worker pool arbitrário.

## Binary loading and runtime attack classes (subagent_type: `general`)

**Library, plugin, and executable search-order trust**
Processos privilegiados carregando bibliotecas dinâmicas (`dlopen`, `LoadLibrary`), plugins ou binários auxiliares por nome relativo influenciável pelo diretório atual, `RPATH`/`LD_LIBRARY_PATH` inseguro ou diretórios graváveis por usuários de menor privilégio (*DLL/SO hijacking*).

**Missing artifact identity or signature binding**
Loaders que verificam assinatura/hash de um arquivo em disco por caminho textual e em seguida mapeiam em memória (`mmap`/`exec`) abrindo novamente o caminho sem prender ao descritor verificado (TOCTOU entre verificação e mapeamento).

**Malformed binary metadata and relocation handling**
Parsers de formatos binários (ELF/PE/Mach-O/WASM/imagens/fontes/PDF) confiando em offsets, contagens de seções, tabelas de relocação ou índices de símbolos antes de validar faixas e sobreposições.

**JIT and generated-code consistency**
Divergências entre validador de bytecode, interpretador e otimizador JIT quanto a tipos, verificação de limites (*bounds-check elimination* incorreta), efeitos colaterais ou tempo de vida de registradores/objetos.

**Unload, reload, and teardown safety**
Ponteiros de função ativos, callbacks registrados ou threads em background sobrevivendo ao descarregamento (`dlclose`) de um módulo dinâmico.

## Kernel and privileged-interface attack classes (subagent_type: `general`)

**User-copy bounds and repeated reads**
Syscalls, handlers `ioctl`, drivers ou servidores de memória compartilhada que leem um cabeçalho/tamanho da memória mutável do usuário, validam-no e depois leem novamente o mesmo endereço de memória do usuário (*double-fetch* race condition) em vez de copiar uma única vez para memória privada do kernel/servidor e validar a cópia imutável.

**Privileged object lifecycle and dispatch consistency**
Objetos expostos via descritores/handles com retenção/liberação desbalanceada em caminhos de erro, destruição sem drenar observadores ou tabelas de despacho com índices sem checagem de limite.

**Under-authorized powerful interfaces**
Dispositivos `/dev/*`, sockets administrativos ou chamadas `ioctl` que validam o formato da struct, mas esquecem de verificar capacidades (`CAP_SYS_ADMIN`, UID/credencial de abertura do descritor, e não apenas do chamador de `read`/`write` para evitar *confused deputy* via stdout redirecionado).

## Universal moves (aplicar transversalmente neste domínio)

- Monte uma tabela para cada fronteira de parser ou FFI: tipo/comprimento aceito, representação validada, dono da alocação, consumidor, afinidade de thread e rotina de liberação — quase toda vulnerabilidade nativa surge de uma divergência nessa tabela.
- Inspecione caminhos irmãos, correções antigas de CVEs e branches de erro: uma checagem adicionada em um caller não protege os demais.
- Use harnesses locais mínimos compilados com `-fsanitize=address,undefined` (ou `-race` / `cargo miri` quando disponível offline) e fixtures de poucos bytes.

## Validation rules (aplicar antes de reportar QUALQUER achado aqui)

1. Estabeleça uma entrada não confiável realista e a operação exata que viola um invariante de limite, tipo, tempo de vida, ABI, concorrência, loader ou autoridade.
2. Classifique precisamente o efeito observado: leitura fora de limites (OOB read), escrita fora de limites (OOB write), use-after-free, confusão de tipo, vazamento de memória não inicializada, carga de biblioteca não autorizada, deadlock ou abort seguro — jamais afirme RCE sem prova, nem eleve um abort seguro além do impacto real demonstrado.
3. Execute o menor harness local ou teste existente com sanitizers para capturar o trace determinístico e registre flags/arquitetura de compilação.
4. Em concorrência/double-fetch, use escalonamento determinístico ou TSan; em loaders, comprove a divergência entre o objeto verificado e o objeto mapeado.
5. Retorne `confirmed` apenas com input exato, trace no código e resultado reproduzido localmente; retorne `needs_validation` para fatos pendentes de ABI externa, flags de build de produção ou alcançabilidade externa.
