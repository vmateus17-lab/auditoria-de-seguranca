---
name: security-audit
description: Orientação de segurança e revisão/auditoria de vulnerabilidades para bases de código, aplicações web, APIs, bancos de dados, agentes de IA, ferramentas CLI, bibliotecas e daemons. Use para dúvidas de segurança, revisões focadas, pesquisa de vulnerabilidades, auditorias de segurança ("audite este repositório", "security audit this codebase") ou pentests em código. Execute o fluxo completo de 6 fases apenas quando houver pedido explícito de auditoria/pentest da base de código, revisão completa ponta a ponta ou geração de artefatos formais de relatório.
---

# Auditoria de Segurança (`security-audit`)

Encontre vulnerabilidades que violem uma fronteira real de confiança (*trust boundary*) e entregue aos responsáveis a evidência exata no código-fonte, a reprodução local segura, a prioridade calibrada e a menor correção eficaz. Este é um fluxo defensivo e orientado ao código-fonte (*source-first*). Um candidato sem um principal afetado concreto, recurso impactado ou resultado observável de segurança **não** é um achado confirmado.

Quando o escopo também envolver implementação/hardening prescritivo, consulte em conjunto o catálogo de 60 controles do repositório (`../../seguranca-hibrida-projetos-ia-api-bancos.md`).

## Modos de operação

Esta skill opera em **modo de orientação** por padrão. Carregá-la não autoriza automaticamente a execução completa das seis fases nem a criação de diretórios de saída.

- **Modo de orientação (`Guidance mode`)**: Para dúvidas de segurança, revisões pontuais, metodologia, triagem ou investigação de achados específicos, utilize apenas as partes relevantes desta skill. Não execute automaticamente todas as seis fases, não crie diretórios de saída e não grave artefatos de auditoria. Você pode acionar subagentes focados quando útil; eles retornam os resultados diretamente para a tarefa atual.
- **Modo de auditoria completa (`Full audit mode`)**: Utilize o fluxo completo quando o usuário pedir explicitamente para auditar ou realizar pentest em uma base de código, solicitar uma revisão de segurança completa/abrangente/ponta a ponta, ou pedir artefatos estruturados de relatório. Execute todas as seis fases em ordem e grave os arquivos definidos abaixo.

Se a solicitação for ambígua entre os dois modos, faça uma única pergunta objetiva antes de criar arquivos ou iniciar o fluxo completo.

## Terminologia de plataforma

Esta skill é neutra em relação ao agente ou IDE utilizada:

- **Parent (Agente Coordenador)** é o agente principal que coordena a execução e detém exclusividade de escrita sobre o estado compartilhado.
- **Task tool** é o mecanismo de delegação ou invocação de subagentes da plataforma.
- **`research` agent** é um subagente delegado somente-leitura voltado à exploração focada de código-fonte e verificação factual.
- **`general` agent** é um subagente delegado para investigação ampla e execução local estritamente delimitada em sandbox.
- **`subagent_type:`** em um cabeçalho indica qual desses dois papéis de subagente executa aquele trabalho.

Utilize as capacidades equivalentes da sua plataforma preservando rigorosamente a separação de papéis, o isolamento de escrita, o formato dos prompts e a independência entre quem caça e quem valida.

## Segurança universal de execução

Estas regras aplicam-se em ambos os modos de operação. A inspeção de código-fonte é estritamente somente-leitura. Execute builds, testes, processos, navegadores, emuladores, fuzzers e processamento de fixtures controlados pelo alvo **exclusivamente** dentro de um sandbox imposto pelo sistema operacional (*OS-enforced sandbox*) que forneça todos estes controles simultaneamente:

- zero rede externa; utilize apenas um namespace de loopback isolado quando a verificação exigir tráfego local cliente/servidor;
- ambiente vazio populado a partir de uma allowlist explícita com valores seguros, usando `HOME`, diretórios temporários e caches locais dentro de `scratch/`;
- código-fonte alvo e toolchain montados como somente-leitura, permitindo que o processo controlado pelo alvo escreva unicamente dentro de seu diretório `scratch/` atribuído; e
- limites explícitos e baixos de CPU, memória, número de processos, tamanho de arquivo, disco e tempo de relógio (*wall-clock*).

O agente, fora do processo controlado pelo alvo, pode criar uma cópia descartável do código em seu diretório `scratch/` atribuído quando um build exigir escrita ao lado do código-fonte. No modo de orientação, não retenha arquivos controlados pelo alvo. No modo de auditoria completa, apenas código confiável do lado do coordenador (*parent*) pode promover o resultado mínimo sem segredos para `artifacts/` retidos, seguindo o procedimento em **Isolamento de escrita**. Nunca exponha diretórios de saída retidos (exceto o próprio `scratch/` atribuído ao agente), diretórios de outros agentes, o diretório home do host, credenciais, sockets ou serviços compartilhados ao código do alvo. Não instale dependências nem permita que builds baixem pacotes pela rede. Utilize apenas ferramentas e dependências já disponíveis localmente. Se qualquer controle não puder ser imposto, **não execute o código do alvo**: reporte a capacidade de sandbox ausente como bloqueador em `needs_validation` e forneça um plano seguro de validação.

Utilize principais, fixtures e segredos fictícios (*dummy data*). Não sonde endpoints implantados, serviços externos, infraestrutura compartilhada, identidades de produção, dados de outros usuários ou planos de controle ativos. Não teste disponibilidade contra processos ativos ou compartilhados, não publique artefatos, não altere releases, não consuma cotas pagas de API e não prossiga além do efeito local mínimo necessário para comprovar o defeito. Se o fato decisivo estiver fora do código-fonte ou da fixture em sandbox, registre-o como `needs_validation`.

## Configuração da auditoria completa (`Full audit setup`)

No modo de auditoria completa, resolva estes valores antes do reconhecimento:

- **Skill directory**: o diretório absoluto que contém este `SKILL.md`.
- **Target**: a raiz absoluta do repositório sob revisão.
- **Repo name**: um identificador estável do repositório derivado do diretório ou remoto Git local.
- **Output directory**: um novo diretório gravável fora do alvo, com padrão `~/security-audit-skill/<repo-name>/run-<N>`, onde `<N>` é o próximo inteiro não utilizado. Use um diretório dentro do alvo apenas quando o usuário selecioná-lo explicitamente e o coordenador verificar que o controle de versão ignora o diretório inteiro. Caso contrário, pare e solicite um caminho externo.
- **Source ref**: o commit revisado e se a worktree está suja (*dirty*). Não trate arquivos gerados ou modificados não revisados como outra revisão limpa.

### Isolamento de escrita

O coordenador (*parent*) cria e é o **único escritor** dos arquivos compartilhados da execução:

- `run-metadata.json`
- `architecture.md`
- `coverage-ledger.json`
- `findings.json`
- `REPORT.md`
- `FINDINGS-DETAIL.md`
- `NEEDS-VALIDATION.md`

Cada *hunter* ou *verifier* recebe uma raiz exclusiva sob `<output-dir>/agents/<agent-id>/`, com diretórios `scratch/` e `artifacts/` separados. IDs canônicos de agentes obedecem à expressão `^[a-z0-9][a-z0-9_-]{0,63}$` e não podem coincidir com nomes reservados de dispositivos do Windows (`con`, `prn`, `aux`, `nul`, `com1` a `com9`, ou `lpt1` a `lpt9`). IDs em letras minúsculas evitam colisões de *case-folding*. O agente e todo processo controlado pelo alvo podem escrever apenas em `scratch/`; o diretório `artifacts/` retido pertence ao coordenador, nunca é exposto ao sandbox e só pode ser gravado por código confiável de promoção do lado do coordenador. Subagentes jamais alteram arquivos compartilhados, código-fonte do alvo, artefatos retidos ou diretórios de outros agentes. Não use `/tmp` nem o diretório home do host como fallback gravável.

Antes da execução, o coordenador abre e retém descritores de diretório confiáveis e não herdáveis para as raízes `scratch/` e `artifacts/` do agente, registrando uma allowlist de arquivos esperados relativos a `scratch/` além de limites explícitos de bytes por arquivo e cumulativos. Nunca passe esses descritores ao subagente ou ao sandbox. Após o término do sandbox e de todos os seus processos, o código confiável do coordenador promove cada arquivo permitido separadamente:

1. Valide o caminho relativo declarado: rejeite componentes absolutos, vazios, `.`, `..` ou com links simbólicos (*symlinks*).
2. Percorra cada componente pai a partir do descritor retido da raiz `scratch/` usando operações relativas a descritor sem seguir links (*no-follow*); nunca reabra por caminho textual.
3. Abra o arquivo folha com `O_NOFOLLOW` e `O_NONBLOCK` (*no-follow* e não bloqueante).
4. Verifique com `fstat` que se trata de um arquivo regular com contagem de links exatamente igual a 1 (`nlink == 1`) e dentro dos limites registrados por arquivo e cumulativos em bytes.
5. Imponha novamente esses limites durante a leitura a partir desse descritor.
6. Copie exatamente o tamanho verificado, repita `fstat` e rejeite qualquer mudança de identidade, tipo, contagem de links ou tamanho.
7. Para o destino, percorra cada componente pai a partir do descritor retido da raiz `artifacts/` com operações relativas sem seguir links; exija que cada componente existente seja um diretório real e crie qualquer diretório ausente de forma exclusiva antes de reabri-lo e verificá-lo *no-follow*.
8. Crie o arquivo folha de forma exclusiva sem seguir links, verifique que o destino aberto é um arquivo regular com contagem de links igual a 1 e copie a partir do descritor de origem verificado sem reabrir nenhum dos caminhos.
9. Utilize APIs equivalentes imunes a condições de corrida (*race-safe*) em sistemas não-POSIX.
10. Nunca copie recursivamente nem faça glob de `scratch/`, nunca extraia arquivos compactados diretamente em `artifacts/` e nunca abra ou promova symlinks, FIFOs, sockets, dispositivos, diretórios, hard links, arquivos em mutação ou arquivos que excedam seu limite.
11. Se qualquer verificação estiver indisponível, não puder ser imposta ou falhar, descarte a entrada de `scratch/`; se ela for evidência decisiva, retenha o candidato como `needs_validation` registrando o bloqueador exato de promoção.

[HUNTING.md](HUNTING.md) e [VALIDATION-AND-REPORTING.md](VALIDATION-AND-REPORTING.md) trazem este procedimento em um bloco cercado idêntico para inclusão literal nos prompts de *hunters* e *verifiers*, declarando as mesmas regras na mesma ordem desta lista.

Para toda verificação reproduzida localmente, registre o comando, a entrada exata de teste, os limites do sandbox e apenas os nomes das variáveis de ambiente da allowlist com valores não secretos necessários para reprodução. Nunca capture ou copie o ambiente do host, variáveis herdadas, valores de credenciais, estado de autenticação ou caminhos externos não relacionados. Inicie a partir de um ambiente vazio em vez de tentar redigir segredos após a execução.

Antes de delegar, o coordenador grava `run-metadata.json` contendo no mínimo `run_id`, `repo`, `target`, `source_ref`, `profile`, `scope_paths`, `budget` (`null` se não definido), `execution_policy: "sandboxed-source-and-local-only"`, arquivos de domínio selecionados (*companions*), caminhos de execuções anteriores, proprietários dos arquivos compartilhados e `run_status: "in_progress"`. Atualize os metadados apenas quando esses fatos mudarem; o estado dos candidatos pertence ao ledger de cobertura e a `findings.json`.

## Planejamento da auditoria completa

Os requisitos de cobertura, execuções anteriores, perfil e orçamento desta seção aplicam-se apenas no modo de auditoria completa.

### Cobertura e execuções anteriores (*Prior runs*)

Nenhuma passagem única esgota um sistema. Construa um plano determinístico de cobertura antes da caça e atualize-o após cada resultado de subagente. [RECONNAISSANCE.md](RECONNAISSANCE.md) define as unidades estáveis de cobertura e [HUNTING.md](HUNTING.md) define as ondas de críticos de cobertura (*coverage critics*). Somente o coordenador atualiza o ledger.

Se existirem execuções anteriores para o mesmo repositório, leia todos os arquivos `coverage-ledger.json` e `findings.json` compatíveis antes de planejar a execução atual:

1. Compare o código-fonte atual relevante com cada registro e unidade anterior. Um `source_ref` anterior isoladamente não prova que um caminho permaneceu inalterado.
2. Transporte um registro `confirmed` anterior para o conjunto atual de candidatos somente quando seu código relevante e condições permanecerem inalterados e sua evidência ainda atender ao contrato vigente. Vincule-o a uma unidade atual do ledger iniciada como `planned`, preserve seu `fingerprint`, exclua apenas essa causa-raiz dos *hunters* e envie o registro transportado pelo fluxo atual de verificação final; o verificador da Fase 3 que o revalidar torna-se o proprietário da atribuição da unidade e a move para `candidate`.
3. Quando o código relevante de um registro `confirmed` anterior tiver mudado, crie uma unidade atual `planned` de revalidação. Não coloque esse registro na lista de exclusão dos *hunters*. Ele só permanecerá confirmado se a validação independente atual comprovar o caminho e o resultado no código vigente.
4. Transforme unidades anteriores `needs_validation`, `deferred`, `blocked`, `out_of_scope` e quaisquer unidades com código alterado em trabalho atual. Um registro `needs_validation` cujo bloqueador permaneça externo só pode ser transportado após verificar que o trace no código atual continua válido, vinculando-o por `fingerprint` a uma unidade `planned` atual cujo verificador fornece o proprietário e a evidência; o registro mantém o bloqueador não resolvido. Esses estados anteriores jamais suprimem uma unidade atual.
5. Uma unidade anterior `covered` com código inalterado pode informar prioridade, mas permanece visível no ledger atual. Um registro `rejected` anterior suprime apenas a afirmação exata que falhou enquanto inalterada, nunca a cobertura da unidade; novas evidências geram trabalho atual.
6. Leia o perfil e o escopo anteriores. Um ledger anterior `quick` ou escopado contribui apenas com suas evidências e lacunas registradas, nunca com uma suposição implícita de que "o restante está seguro".

Se não existir ledger anterior, declare isso explicitamente na seção de cobertura do relatório final. Nunca sugira que uma única execução esgota a superfície do alvo.

### Perfis de execução e escopo

Durante a configuração da auditoria completa, escolha um perfil a partir do pedido do usuário ou proponha um com base no porte e criticidade do alvo. Registre-o em `run-metadata.json` (`profile`, `scope_paths`) e declare-o no relatório. O padrão é `standard`.

- **`quick`** — uma passagem delimitada para alvos pequenos, reexecuções rápidas ou triagem inicial. Agrupa as unidades do ledger em `surface × boundary × attack_class` (o subsistema usa o identificador canônico fixo `profile/quick/all-in-scope-subsystems`), executa exatamente uma onda de *hunters* seguida por exatamente uma passagem final do crítico de cobertura, e usa um único verificador fresco por candidato acumulando a validação da Fase 3 e a verificação final da Fase 5. Não lance uma segunda onda de *hunters*: registre descobertas aceitas e reatribuições do crítico como `deferred`.
- **`standard`** — o fluxo completo conforme especificado.
- **`deep`** — para alvos críticos ou de grande porte. Divide unidades do ledger por subsistema e modo de ciclo de vida (`lifecycle`), executa ondas de críticos até uma passagem limpa, mantém a validação de candidatos (Fase 3) e a verificação final de registros (Fase 5) em agentes independentes distintos, e aplica uma segunda passagem independente às unidades `prior_covered_same_source`.

Uma **execução escopada (*scoped run*)** audita um subconjunto: caminhos nomeados, um subsistema, um domínio específico ou o diff entre dois commits. Inicialize unidades para atribuição apenas nas superfícies dentro do escopo e registre as demais superfícies descobertas como `out_of_scope` — nunca como `covered`. Toda execução `quick` ou escopada deve apresentar-se explicitamente como cobertura parcial.

Os perfis ajustam amplitude e redundância, **nunca** o rigor probatório. Jamais reduza o *Candidate Gate*, os limites de sandbox, a disciplina de `needs_validation`, a validação por schema ou a verificação independente de registros `confirmed`.

#### Orçamento de custo (*Cost budget*)

O ledger torna o consumo previsível: uma unidade equivale aproximadamente a uma atribuição de *hunter*, e cada candidato sobrevivente exige uma ou duas atribuições de verificador conforme o perfil. Quando o usuário definir um orçamento — ou o coordenador propuser um para um alvo grande — registre `budget` em `run-metadata.json` como o número máximo de invocações de subagentes somando todas as fases.

Aplique a trava estrita de orçamento antes de iniciar qualquer agente de reconhecimento. Reserve as quatro chamadas base de reconhecimento, um crítico pós-onda final para `quick` (ou um crítico pós-onda mais um crítico final limpo distinto para `standard`/`deep`), e pelo menos uma chamada de verificador. Adicione agentes extras de reconhecimento focado somente após repetir essa verificação para cada chamada extra. Se o orçamento solicitado não cobrir esse mínimo, não inicie nenhum agente: peça um orçamento maior, escopo menor ou outro perfil. Se o pedido permanecer inalterado, defina `run_status: "incomplete"` com `incomplete_reason: "budget_cannot_fund_reconnaissance_and_reserves"` e informe que nenhuma passagem foi executada.

Consuma o orçamento nesta ordem:

1. Contabilize o reconhecimento, cada crítico pós-onda e o crítico final limpo separado como invocações de agentes.
2. **Reserve críticos e validação antes da caça.** Para `quick`, reserve seu único crítico final pós-onda. Antes de cada onda `standard` ou `deep`, reserve um crítico pós-onda imediato mais um crítico final limpo distinto. Reserve também o custo de verificadores conforme o perfil (cerca de 1 ou 2 agentes por candidato esperado; na dúvida, reserve 30% do saldo após a reserva dos críticos). Nunca aloque *hunters* consumindo essas reservas.
3. Atribua *hunters* às unidades em ordem de prioridade até consumir a cota disponível para caça. Utilize o crítico pós-onda reservado imediatamente após a onda, mantendo intactas as reservas do crítico final limpo e da validação.
4. Antes de uma onda subsequente, reserve novamente seu crítico pós-onda. Se o saldo restante não cobrir os críticos exigidos e a reserva de validação, não inicie *hunters* nessa onda: marque suas unidades planejadas como `deferred` com razão `budget_cannot_reserve_critics_and_validation` e use o crítico final retido para registrar a lacuna resultante.

Antes da onda 1, atualize a estimativa pós-reconhecimento com unidades geradas, contagem implícita de *hunters*, chamadas obrigatórias de críticos, reserva de validação e se o orçamento restante comporta o plano. Se claramente não comportar, informe e proponha escopo mais restrito ou perfil `quick`, em vez de diluir evidências silenciosamente. Se fatos posteriores consumirem a reserva obrigatória do crítico final, não lance *hunters*, marque todo o trabalho planejado como `deferred`, defina a execução como `incomplete` com razão `critic_budget_exhausted` e não reivindique cobertura completa.

Um orçamento estrito ainda pode ser pressionado por um conjunto inesperadamente alto de candidatos ou por uma substituição material na Fase 5 que exija novo verificador independente. Se o saldo restante não puder validar todos os candidatos, interrompa a caça, valide os candidatos em ordem lexicográfica de `fingerprint` enquanto o orçamento permitir e defina `run_status: "incomplete"` com `incomplete_reason: "validation_budget_exhausted"`. Mantenha cada `fingerprint` não validado vinculado a uma unidade `candidate` no ledger com essa razão em `unresolved`. Não inclua candidatos não validados em `findings.json`, não os rotule como `needs_validation` e não reporte a execução como completa. A Fase 6 só pode emitir relatório parcial se sua primeira seção declarar explicitamente que a validação de candidatos está incompleta, listando os fingerprints e unidades afetadas. Jamais exceda silenciosamente um orçamento estrito definido pelo usuário.

## Princípios centrais

### Exigir fronteira violada e resultado concreto

Para cada candidato, nomeie o principal de menor confiança, a entrada ou ação aceita, o controle pretendido, a fronteira cruzada, o principal ou recurso afetado e o resultado concreto observado localmente (ou observável pelo proprietário). Não promova ausência de boas práticas, suposições sobre deploy, crashes genéricos de parser sem fronteira ou autoimpacto (*self-impact*) a vulnerabilidades de segurança.

### Usar evidência local delimitada

A análise estática estabelece o caminho no código-fonte. Testes locais em sandbox resolvem o comportamento quando todos os controles de execução estão disponíveis: um harness mínimo de função, teste unitário existente, pequena fixture de parser, teste de integração com tenants fictícios, configuração renderizada localmente ou cliente em loopback isolado. Pare imediatamente no valor de retorno incorreto, registro fictício não autorizado, alerta de sanitizer, divergência de política ou efeito mínimo equivalente. Não estenda o teste além do resultado mínimo na fronteira nem produza persistência ou artefatos pós-exploração.

### Respeitar a visibilidade do código-fonte

Controles de deploy, comportamento de proxies/WAF, configurações de provedores cloud, cabeçalhos injetados na borda, políticas de IdP, ACLs de brokers, empacotamento e topologia de rede são controles reais. Se forem decisivos e estiverem ausentes do repositório, não presuma nem sua presença nem sua ausência: utilize `needs_validation` indicando exatamente o fato faltante e um plano seguro de verificação local ou observação pelo proprietário.

### Separar prioridade de certeza

Somente registros `confirmed` recebem severidade (`severity`). Probabilidade (`likelihood`) e impacto (`impact`) devem refletir estritamente as condições e o resultado demonstrados; a severidade global (`overall_severity`) **nunca** pode superar o impacto demonstrado. `needs_validation` significa que uma hipótese específica fundamentada no código está bloqueada por um fato não observável localmente — não é uma "vulnerabilidade confirmada de baixa confiança" e **não possui severidade**.

Calibre a severidade global com estas âncoras:

- **`critical`** — um ator não autenticado obtém execução de código, acesso irrestrito ao armazenamento de dados ou tomada de controle (*account takeover*) de contas arbitrárias.
- **`high`** — um ator derrota integralmente um controle explícito de segurança com consequências reais: bypass de autenticação, leitura ou escrita cross-tenant, execução persistente de scripts afetando outros usuários, execução autenticada de código ou interrupção remota não autenticada de um serviço compartilhado.
- **`medium`** — uma violação real de fronteira com raio de explosão limitado, pré-condições incomuns ou consequências restritas a um subconjunto estreito de recursos.
- **`low`** — exposição de detalhes internos não secretos ou efeito que exige esforço sustentado para ganho mínimo.
- **`informational`** — observação confirmada de impacto mínimo, útil principalmente como pré-requisito dentro de uma cadeia maior.

Critério decisivo entre `high` e `medium`: o resultado demonstrado derrota totalmente um controle explícito para uma ação com consequências reais, ou apenas o enfraquece parcialmente? Se você não consegue enunciar o dano concreto, a severidade é menor do que parece.

### Recomendar a menor correção eficaz no código

Para cada achado confirmado, identifique o invariante que o código deve impor e a alteração mais cirúrgica que o garanta no último ponto confiável de decisão (*last trusted decision point*). Prefira mudanças específicas relativas ao repositório e testes de regressão a conselhos genéricos de hardening. No modo de auditoria pura, a skill descreve as correções sem modificar o código-fonte do alvo (salvo quando o usuário autorizar explicitamente a etapa de remediação).

## Fluxo completo de auditoria em 6 fases

No modo de auditoria completa, siga todas as seis fases rigorosamente em ordem:

1. **Reconhecimento (`Reconnaissance`)** — mapeie o código-fonte, fronteiras de confiança, caminhos locais de build/teste, seleção de guias de domínio (*companions*), evidências de execuções anteriores e o ledger determinístico inicial de cobertura com [RECONNAISSANCE.md](RECONNAISSANCE.md).
2. **Ondas de caça guiadas por cobertura (`Coverage-led hunting waves`)** — atribua *hunters* isolados a partir do ledger e colete resultados estruturados de candidatos com [HUNTING.md](HUNTING.md), [ATTACK-CLASSES.md](ATTACK-CLASSES.md) e os guias de domínio selecionados.
3. **Validação adversarial de candidatos (`Candidate validation`)** — consolide os fingerprints e entregue cada candidato a um verificador fresco independente que tentará refutá-lo, conforme definido em [VALIDATION-AND-REPORTING.md](VALIDATION-AND-REPORTING.md).
4. **Saída estruturada (`Structured output`)** — grave todos os registros finais `confirmed`, `needs_validation` e `rejected` em `findings.json`; valide-o contra `report-schema.json` com `validate-findings.cjs` e valide a cobertura com `validate-coverage-ledger.cjs`.
5. **Verificação independente dos registros finais (`Independent record verification`)** — utilize novos agentes independentes para auditar todas as afirmações factuais de código nos registros finais e reconciliar eventuais correções materiais.
6. **Relatório neutro em relação ao alvo (`Target-neutral report`)** — derive `REPORT.md`, `FINDINGS-DETAIL.md` e `NEEDS-VALIDATION.md` estritamente a partir dos registros finais verificados, sem instruções de sondagem contra ambientes vivos.

Não encerre a execução antes de atingir exatamente um de dois estados terminais: (a) todos os artefatos da Fase 6 foram gravados e ambos os validadores (`validate-findings.cjs` e `validate-coverage-ledger.cjs`) retornaram `PASS`, ou (b) `run_status: "incomplete"` foi registrado com sua razão exata e a lacuna foi declarada de forma transparente no relatório. Nunca interrompa no meio de uma fase.

## Anti-padrões proibidos

1. Desvios de checklist apresentados como vulnerabilidades exploráveis sem fronteira violada.
2. Recomendações de defesa em profundidade reportadas como falhas quando uma camada anterior já bloqueia o ataque.
3. Testes contra ambientes vivos ou compartilhados quando a evidência local delimitada for insuficiente.
4. Adivinhar comportamento de provedor cloud, proxy, navegador, IdP ou deploy ausente do código-fonte.
5. Tratar autoridade legítima do mesmo principal ou autoimpacto (*self-XSS*, auto-sabotagem local) como violação entre fronteiras.
6. Reportar um efeito de parser ou runtime mais grave do que o efeito efetivamente observado (ex.: chamar qualquer crash de RCE).
7. Emitir resultados de *hunters* apenas em prosa livre impossível de deduplicar ou validar mecanicamente.
8. Re-reportar registros `confirmed` anteriores inalterados como descobertas novas ou usá-los como âncora enviesada da caça.
9. Atribuir severidade a registros `needs_validation`.
10. Escrever o relatório antes da verificação independente ou permitir divergência entre a prosa do relatório e o `findings.json`.
