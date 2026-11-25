# Reconhecimento (`Reconnaissance`)

### Fase 1: Mapear o código-fonte e planejar a cobertura determinística

O agente coordenador (*parent*) inicializa `run-metadata.json`, aplica a trava estrita de orçamento pré-reconhecimento definida em `SKILL.md` e cria os diretórios `scratch/` dos agentes e o ledger compartilhado antes de iniciar a caça. Se a trava de orçamento falhar, registre o status incompleto nos metadados e não inicie nenhum agente de reconhecimento. O reconhecimento lê exclusivamente o repositório-alvo e o estado de build/configuração disponível localmente — sem contatar endpoints em produção/staging, provedores externos de identidade, registries, brokers, APIs de nuvem ou serviços compartilhados.

Inicie os subagentes `research` em paralelo. Eles retornam fatos estruturados ao coordenador e **não gravam arquivos**.

**Agente 1a: Produto, stack e operação local**

```text
Leia o alvo em <target>. Não utilize acesso à rede. Retorne:
1. Tipo de produto, perfis de usuários, operadores e ações cotidianas sensíveis à confiança.
2. Linguagens, frameworks, sistema de build, runtimes e modelos de implantação visíveis localmente.
3. Pontos de entrada relativos ao repositório e fronteiras entre subsistemas.
4. Comandos exatos de build e teste que poderiam rodar offline com dependências locais, seus locais esperados de escrita e as entradas controladas pelo alvo que processam. Não os execute durante o reconhecimento.
5. Software ou protocolo comparável visível na documentação local e nas dependências. Se nenhuma comparação útil estiver fundamentada no código, informe explicitamente.
6. Toolchains locais ausentes ou fatos de runtime que limitem a execução delimitada em sandbox.
Retorne apenas fatos fundamentados no código com referências relativas ao repositório no formato arquivo:linha.
```

**Agente 1b: Principais, autoridade e controles**

```text
Leia todo o código que estabelece identidade, autorização, isolamento e privilégio. Mapeie:
1. Cada principal de menor confiança e as ações que ele possui por design.
2. Autenticação ou identidade de par (peer identity) em cada superfície de entrada.
3. Autorização por recurso e escopo de tenant/proprietário (BOLA/BFLA/RLS).
4. Autoridade de processo, navegador, workload, pipeline CI, plugin, modelo/ferramenta de IA, dispositivo ou IPC local.
5. Mudanças de privilégio, confirmação humana (HITL/step-up), revogação, recuperação de conta e caminhos de fallback.
6. Quais controles são visíveis no código-fonte e quais dependem de um fato de implantação não observável localmente.
Retorne fronteiras de confiança e localizações de controles com referências relativas arquivo:linha. Não infira alcançabilidade em produção.
```

**Agente 1c: Superfícies de entrada, cópias derivadas e sinks**

```text
Inventarie todos os pontos visíveis no código onde entradas externas ou de menor confiança ingressam:
- HTTP/navegador, RPC/mensageria/protocolo, arquivos/compactados/documentos, CLI/env/config, plugins/dependências/CI, eventos cloud/seletores IAM, contexto de modelos de IA/argumentos de ferramentas/MCP, mobile/deep-links/webview e IPC local.
Para cada superfície, siga as principais transformações, cópias armazenadas ou derivadas (caches, índices, vetores, filas, logs) e sinks sensíveis à segurança. Registre limites visíveis no código e caminhos paralelos que levam ao mesmo efeito.
Retorne caminhos relativos ao repositório e números de linha. Seja exaustivo, mas não execute comandos nem envie cargas.
```

**Agente 1d: Execução local e visibilidade de implantação**

```text
Leia testes, definições de build, manifestos, empacotamento e overlays de ambiente mantidos no repositório. Retorne:
1. Pequenos testes offline ou fixtures existentes capazes de validar fronteiras de confiança com dados fictícios dentro do sandbox obrigatório do SO.
2. Processos que possam usar um namespace de rede loopback isolado sem dependências externas ou compartilhadas.
3. Comandos que baixariam dependências, publicariam artefatos, acionariam APIs pagas/provedores externos ou afetariam estado compartilhado; marque-os como proibidos para esta execução.
4. Controles de implantação e anexos (WAF, proxy, IAM, políticas de banco/broker) que o código-fonte não pode comprovar sozinho e que exigirão needs_validation se forem decisivos.
5. O caminho de código ativo final para cada modo de deploy apenas onde o repositório o seleciona deterministicamente.
6. Se a plataforma local consegue impor ambiente vazio com allowlist, ausência de rede externa, montagem somente-leitura do alvo/toolchain, escrita restrita a scratch/ e limites explícitos de CPU, memória, processos, tamanho de arquivo, disco e wall-clock. Controles ausentes bloqueiam a execução de código controlado pelo alvo.
7. Se o código confiável do coordenador consegue promover arquivos pré-declarados de scratch/ usando travessia de descritores sem seguir links (O_NOFOLLOW), checagens não bloqueantes de arquivo regular, criação exclusiva no destino e limites explícitos de tamanho por arquivo e cumulativo. Ausência desses controles bloqueia o uso de arquivos de scratch/ como evidência.
```

Adicione agentes focados de reconhecimento para modos de implantação ou subsistemas materialmente distintos que os quatro agentes base não cubram. Nunca os omita silenciosamente: se a trava de orçamento em `SKILL.md` impedir um agente focado, não inicie nada para ele, registre a área não mapeada como unidade `deferred` no ledger com razão `budget_cannot_reserve_critics_and_validation` e declare a lacuna no relatório.

## Entrada de execuções anteriores (`Prior-run input`)

Antes de selecionar o trabalho, o coordenador lê todos os arquivos `coverage-ledger.json` e `findings.json` anteriores disponíveis para o mesmo repositório:

- Compare localizações de código, controles, condições e identidade derivada do código de cada registro e unidade anterior contra o código atual.
- Transporte um registro `confirmed` anterior inalterado para o conjunto atual de candidatos, mantendo o mesmo `fingerprint`, somente quando o código relevante, as condições e as evidências qualificadoras continuarem válidos. Vincule-o a uma unidade atual `planned` com `prior_status: "prior_confirmed_same_source"` e inclua apenas essa causa-raiz na lista de exclusão dos *hunters*. O verificador da Fase 3 que revalidar o registro transportado torna-se o proprietário da unidade; sua rechecagem de código constitui o primeiro check da unidade e a move para `candidate` com o fingerprint transportado.
- Crie uma unidade planejada `prior_confirmed_changed_source` de revalidação quando qualquer trecho relevante ou condição tiver mudado. Não exclua essa causa-raiz da caça nem assuma que o veredito anterior permanece válido.
- Crie unidades de trabalho atuais para toda unidade anterior `needs_validation`, `deferred`, `blocked`, `out_of_scope` e unidades com código alterado. Esses estados servem como entrada de priorização, nunca como chaves de supressão.
- Transporte um registro `needs_validation` anterior ainda bloqueado mantendo o mesmo `fingerprint` apenas após confirmar que o código atual sustenta seu trace. Vincule-o a uma unidade `planned` atual com `prior_status: "prior_needs_validation"`; o registro conserva o bloqueador pendente. O verificador da Fase 3 que o revisar torna-se proprietário da unidade e a move para `candidate`, incluindo o registro na verificação final.
- Trate registros `rejected` anteriores como afirmações superadas, a menos que novas evidências alterem o trace refutado ou a condição ausente. Uma rejeição inalterada suprime apenas aquela alegação exata, não a revisão da unidade de cobertura.
- Registre ledgers ausentes ou incompatíveis em vez de tratá-los como cobertura vazia.

Declare os caminhos e `source_ref` utilizados em `run-metadata.json`. Resuma apenas as consequências de cobertura em `architecture.md`.

## Resumo arquitetural e seleção de domínios (`companions`)

O coordenador sintetiza `<output-dir>/architecture.md`, com teto rígido de aproximadamente 1.000 palavras, incluindo:

1. Produto, principais, autoridade normal e recursos protegidos.
2. Linha de base de software comparável obtida pelo Agente 1a (quando fundamentada no código): quais trade-offs de segurança o software comparável aceita. Use isso apenas para calibrar esforço e severidade, nunca para descartar uma falha demonstrada; se o software comparável compartilhar um padrão de defeito historicamente explorável, isso reforça o achado. Omita este item quando não houver comparável relevante.
3. Stack tecnológica, caminhos de deploy visíveis no código e limites de build/teste offline.
4. Superfícies de entrada e os principais fluxos *source-to-sink* ou de ciclo de vida.
5. Fronteiras de confiança e o controle visível mais forte em cada uma.
6. Caminhos iniciais relativos ao repositório (`starting_paths`).
7. Lacunas de cobertura anteriores, alvos de revalidação por mudança de código ou bloqueio e exclusões confirmadas de mesma origem.
8. Resumo conciso da seleção de guias de domínio derivada de [ATTACK-CLASSES.md](ATTACK-CLASSES.md): arquivos selecionados e as fronteiras visíveis no código que exigem cada um deles.

Mantenha o bloco ordinário de ataque, os blocos selecionados de domínios e os blocos excluídos com justificativa dentro de cada unidade do `coverage-ledger.json`, e não em `architecture.md`. Isso preserva o limite de 1.000 palavras em repositórios grandes e torna o mapa de prompts dos *hunters* auditável por máquina.

Não selecione um arquivo de domínio apenas porque o nome de uma linguagem ou biblioteca aparece no `package.json` ou `Cargo.toml`. Selecione-o porque o reconhecimento identificou a fronteira sensível à confiança descrita na seção `When to use this file` daquele guia. Não exclua uma fronteira visível apenas porque outro agente revisará uma classe relacionada.

## Ledger determinístico de cobertura (`coverage-ledger.json`)

O coordenador grava `<output-dir>/coverage-ledger.json` como um array JSON de nível superior. Derive uma unidade para cada combinação material de superfície de entrada (`surface`), fronteira de confiança (`boundary`), subsistema (`subsystem`) e classe de ataque ordinária ou de domínio aplicável (`attack_class`), na granularidade definida pelo perfil da execução (`quick` utiliza uma identidade única de subsistema em escopo; `deep` adiciona modos de `lifecycle`). Em execuções escopadas, inicialize para atribuição as superfícies em escopo e mantenha as superfícies excluídas descobertas como unidades `out_of_scope` para que auditorias futuras possam convertê-las em trabalho ativo.

Cada dimensão possui um rótulo legível humano e um valor estável derivado do código em `canonical_refs`. Use exatamente a mesma referência canônica para o mesmo objeto de código entre execuções distintas, mesmo que o rótulo descritivo mude. Boas referências incluem caminho relativo do ponto de entrada mais o símbolo exportado, rota ou identificador de mensagem definido no código, função/middleware que define a fronteira, caminho do pacote no monorepo e referência exata ao bloco de classe de ataque. Uma referência de bloco tem o formato `FILE.md#` seguido do nome exato da classe conforme escrito em negrito ou cabeçalho naquele arquivo (ex.: `ATTACK-CLASSES.md#Access control`, `AI-AND-LLM.md#Core discipline`) — um identificador textual estável, não uma âncora HTML slugificada. Para blocos de seção de guias de domínio, use o texto do cabeçalho antes de qualquer qualificador entre parênteses (por exemplo, `Core discipline`, `Universal moves`, `Validation rules`). Nunca derive referências convertendo rótulos de exibição para minúsculas ou slugs com perda de informação.

Derive `coverage_id` sem colisões:

1. Exija que cada referência esteja em Unicode NFC com valores escalares válidos, conteúdo visível, sem caracteres de controle, formatação, separadores de linha/parágrafo ou code points ignoráveis por padrão, e sem espaços nas extremidades.
2. Codifique seus bytes UTF-8 segundo RFC 3986 (*percent-encoding*): mantenha sem escape apenas `A-Z a-z 0-9 - . _ ~` e use `%HH` em maiúsculas para qualquer outro byte.
3. Una as referências codificadas de `surface`, `boundary`, `subsystem` e `attack_class` com `::`; anexe `lifecycle` codificado ao final quando presente.

Use o valor canônico fixo `profile/quick/all-in-scope-subsystems` na dimensão de subsistema do perfil `quick`. Não inclua número de onda, ID de agente, veredito, severidade ou número de linha em referências canônicas ou no `coverage_id`. Ordene as unidades lexicograficamente por `coverage_id` antes de cada atribuição. Falhe imediatamente diante de qualquer ID duplicado. Se IDs duplicados tiverem campos semânticos distintos, trate como colisão de identidade canônica; jamais faça merge silencioso. O validador também rejeita uma mesma tupla semântica representada por referências canônicas diferentes.

Cada unidade registra:

```json
{
  "coverage_id": "...",
  "canonical_refs": {
    "surface": "src/router.ts#POST /users/:id",
    "boundary": "src/authz.ts#requireOwner",
    "subsystem": "packages/api",
    "attack_class": "ATTACK-CLASSES.md#Access control"
  },
  "surface": "...",
  "boundary": "...",
  "subsystem": "...",
  "attack_class": "...",
  "starting_paths": ["repo/relative/path"],
  "ordinary_attack_class_block": "ATTACK-CLASSES.md#Access control",
  "selected_companion_blocks": ["FILE.md#section"],
  "excluded_blocks": [{"block": "FILE.md#section", "reason": "..."}],
  "prior_status": "new|prior_confirmed_same_source|prior_confirmed_changed_source|prior_needs_validation|prior_deferred|prior_blocked|prior_out_of_scope|prior_covered_same_source|prior_covered_changed_source|prior_rejected_claim_changed|none",
  "attempts": [],
  "wave": 1,
  "status": "planned",
  "agent_id": null,
  "reviewed_paths": [],
  "local_checks": [],
  "result_fingerprints": [],
  "unresolved": []
}
```

Quando `lifecycle` for relevante, adicione tanto `canonical_refs.lifecycle` quanto o campo legível `lifecycle`. `ordinary_attack_class_block` só é `null` quando nenhum bloco ordinário de `ATTACK-CLASSES.md` se aplica. A lista `selected_companion_blocks` inclui cada subclasse aplicável mais os blocos obrigatórios `Core discipline`, `Universal moves` e `Validation rules` do respectivo guia; `excluded_blocks` registra cada bloco considerado mas descartado, junto ao fato do código-fonte que justifica a exclusão.

O coordenador pode adicionar campos auxiliares de controle, mantendo intactos os campos semânticos acima. Em `prior_status`, `new` marca uma superfície vista pela primeira vez quando existem ledgers anteriores compatíveis; `none` marca unidades geradas quando não há ledger anterior compatível.

`attempts` é um arquivo histórico somente-adição (*append-only*) para atribuições com evidências que um crítico de cobertura reabrir. Antes da reatribuição, anexe em `attempts` os valores exatos anteriores de `wave`, `status`, `agent_id`, `reviewed_paths`, `local_checks`, `result_fingerprints` e `unresolved`, acrescidos de `reassignment_reason` fundamentado no código pelo crítico. Apenas estados `blocked`, `covered` e `candidate` podem ser arquivados. Tentativas arquivadas mantêm os mesmos invariantes de estado e evidência das unidades ativas, usam ondas estritamente crescentes abaixo da onda atual e preservam seus proprietários e artefatos originais. A próxima atribuição incrementa `wave`, usa um novo `agent_id` inédito e inicia com evidências ativas vazias. Se o perfil ou orçamento impedir nova atribuição, incremente `wave` e use o estado ativo `deferred` com `agent_id: null`, evidências vazias e a razão de parada em `unresolved`. Jamais copie checks ou artefatos de um proprietário arquivado para o estado ativo.

Imponha exatamente esta tabela de estados:

| Status | `agent_id` da unidade | `reviewed_paths` / `local_checks` | `result_fingerprints` | `unresolved` |
|---|---|---|---|---|
| `planned` | `null` | vazios | vazio | vazio |
| `not_applicable`, `out_of_scope`, `deferred` | `null` | vazios | vazio | razão não vazia |
| `in_progress` | proprietário canônico | vazios | vazio | vazio |
| `blocked` | proprietário canônico | ambos não vazios (evidência parcial própria) | vazio | bloqueador não vazio |
| `covered` | proprietário canônico | ambos não vazios | vazio | vazio |
| `candidate` | proprietário canônico | ambos não vazios | não vazio | opcional |

IDs canônicos de agentes seguem `^[a-z0-9][a-z0-9_-]{0,63}$` e não usam nomes reservados do Windows. Cada check dentro de `local_checks` registra seu próprio `agent_id` e `reviewed_paths` não vazio; o `reviewed_paths` agregado da unidade é exatamente a união dos `reviewed_paths` de seus checks. Um check puramente estático (`method: "source"`) usa `artifact: null`. Um check dinâmico (`method: "local"`) exige um arquivo regular promovido exclusivamente pelo coordenador sob `agents/<check.agent_id>/artifacts/`. Isso permite que evidências do *hunter* e do *verifier* coexistam na mesma unidade sem perda de autoria.

O ledger **é** a prova de cobertura. Um resumo de arquitetura, número de agentes executados ou uma frase genérica como "autenticação revisada" não constitui evidência de cobertura.

Execute `node <skill-dir>/validate-coverage-ledger.cjs <output-dir>/coverage-ledger.json` imediatamente após criar o ledger inicial, após cada atualização feita pelo coordenador e antes da Fase 6. Corrija todos os erros antes de atribuir trabalho ou declarar cobertura.
