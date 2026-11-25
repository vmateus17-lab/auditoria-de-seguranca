# Caça a Vulnerabilidades (`Vulnerability Hunting`)

### Fase 2: Executar ondas de caça guiadas por cobertura

O coordenador (*parent*) atribui unidades `planned` do ledger a subagentes `general` (*hunters*). Utilize *hunters* focados em quantidade suficiente para cobrir as unidades sem misturar fronteiras desconexas. Um mesmo *hunter* pode assumir unidades intimamente relacionadas dentro de um único subsistema; nenhuma unidade pode ficar sem atribuição silenciosamente por limite de agentes — toda unidade que o orçamento não alcançar deve ser marcada explicitamente como `deferred` com razão `budget_cannot_reserve_critics_and_validation`.

Quando o orçamento ou perfil limitar a quantidade de *hunters*, atribua as unidades em ordem estrita de prioridade: (1) superfícies de entrada não autenticadas ou de menor confiança antes das autenticadas; (2) fronteiras que protegem os ativos mais críticos (credenciais, dados cross-tenant, execução de código, autoridade de release); (3) lacunas de execuções anteriores, alvos de revalidação e código alterado antes de repassagens em código inalterado; (4) classes historicamente mais propensas a falhas confirmadas naquele tipo de arquitetura. Empates são resolvidos lexicograficamente por `coverage_id` para manter o determinismo.

Antes do disparo, o coordenador muda as unidades atribuídas para `in_progress`, define um `agent_id` canônico em minúsculas e cria os diretórios `scratch/` e `artifacts/` (este último exclusivo do coordenador). Os *hunters* leem o código-fonte e o contexto fornecido, escrevem unicamente em seu `scratch/` exclusivo e retornam um único objeto JSON estruturado. Eles jamais gravam artefatos retidos nem editam o código do alvo, `architecture.md`, `coverage-ledger.json`, `findings.json` ou arquivos de outros agentes.

## Estrutura obrigatória do prompt do Hunter

Todo prompt enviado a um *hunter* deve conter estas nove partes exatamente nesta ordem:

1. Um preâmbulo de papel de duas frases: o objetivo do *hunter* é encontrar quebras de invariantes de segurança fundamentadas no código-fonte dentro das unidades atribuídas, devendo retornar exatamente um objeto JSON aderente ao contrato de resultado estruturado ao final do prompt.
2. O conteúdo integral de `architecture.md`.
3. Os `coverage_id` atribuídos, subsistema, fronteira, `starting_paths` relativos ao repositório e o mapa de blocos de cada unidade extraído de `coverage-ledger.json`.
4. Os blocos exatos selecionados, copiados literalmente: cada bloco ordinário selecionado de `ATTACK-CLASSES.md` e, de cada guia de domínio selecionado (*companion*), seu `Core discipline`, cada subclasse de ataque escolhida, `Universal moves` e `Validation rules`. Nunca envie apenas os nomes dos arquivos ou das classes.
5. Os blocos ordinários e de domínio explicitamente excluídos acompanhados da razão de cada exclusão.
6. O bloco **Método central de caça** abaixo, seguido literalmente pelo bloco **Procedimento de promoção de artefatos**.
7. O bloco **Regras centrais de validação (`Candidate gate`)** abaixo.
8. Exclusões transportadas de achados `confirmed` anteriores com mesmo código (limitadas a `fingerprint`, título e causa-raiz), além dos `coverage_id` atribuídos a outros *hunters* paralelos que este agente não deve duplicar.
9. Os caminhos exclusivos de `scratch/` e `artifacts/`, o `agent_id` seguro, a allowlist pré-declarada de promoção com limites em bytes e o contrato de saída estruturada, incluindo o bloco JSON abaixo e as ramificações `confirmed` e `needs_validation` de `report-schema.json` copiadas integralmente.

Se uma fronteira inesperada surgir durante a leitura, o *hunter* deve retorná-la em `uncovered` para que o coordenador crie uma unidade estável no ledger e a atribua na próxima onda.

#### Método central de caça — incluir em todo prompt de hunter

```text
## Método defensivo de descoberta de vulnerabilidades

Seu objetivo é encontrar falhas em invariantes de segurança fundamentadas no código-fonte e a menor correção eficaz,
não expandir danos além do resultado na fronteira. Permaneça restrito à revisão de código e execução local delimitada.
Não contate endpoints implantados, APIs de provedores, registries, sistemas de identidade,
message brokers, serviços compartilhados ou outros usuários. Use exclusivamente dados locais fictícios (dummy data).

LEIA O CÓDIGO EM PROFUNDIDADE. Siga cada entrada atribuída através de parsing, identidade,
autorização, normalização, estado, cópias derivadas e o sink final. Leia caminhos irmãos,
legados, em lote (batch), retentativas, cancelamento, migração e tratamento de erros que produzam o mesmo
efeito. Compare controles irmãos quanto à equivalência semântica — não apenas presença — e confronte o que
um componente garante com o que o próximo componente presume.

TRABALHE A PARTIR DE UM INVARIANTE CONCRETO:
1. Nomeie o principal de menor confiança e sua capacidade inicial.
2. Nomeie o valor aceito, ação, transição de estado ou seletor de recurso.
3. Localize o controle que deveria rejeitar, vincular, isolar, limitar ou revogar a operação.
4. Rastreie o caminho exato no código-fonte após essa decisão.
5. Pare no menor registro fictício afetado, valor de retorno incorreto, efeito de integridade de processo
   ou efeito localmente observável sobre recurso compartilhado.
6. Enuncie a alteração em nível de código e o caso de teste de regressão que impõem o invariante.

LIMITE DE PROFUNDIDADE: rastreie apenas caminhos capazes de alcançar sua fronteira atribuída ou cujas
garantias sustentem essa fronteira. Encerre uma linha de investigação assim que o invariante for decidido
positiva ou negativamente, registrando o resultado na saída estruturada — covered, candidate, blocked
ou uma entrada em uncovered — em vez de continuar vagando pelo código.

TESTE CAMINHOS TRISTES E DIVERGÊNCIAS. Verifique valores ausentes, vazios, zero, negativos, máximos,
acima do limite, duplicados, codificações mistas, estados obsoletos, revogados, reordenados, concorrentes,
parcialmente migrados, falhas de dependência e rollback onde a interface os aceitar. Compare canonicalização
e unidades de medida em cada passagem entre parser e política. Em problemas de múltiplos passos, trate cada saída
como pré-requisito e não presuma falha em fronteira posterior; se um pré-requisito não for comprovado, registre bloqueador.

Quando um candidato high ou critical revelar uma causa-raiz reutilizável, procure nos caminhos pertencentes
aos seus coverage_ids atribuídos por variantes léxicas, estruturais e lógicas. Consolide a mesma causa-raiz,
mas comprove condições e impacto de cada variante independentemente. Não investigue unidades de outros agentes;
retorne variantes fora das suas unidades em uncovered.

USE O TESTE LOCAL MAIS ESTREITO QUE DECIDA A QUESTÃO. Builds, testes, processos, navegadores, emuladores,
fuzzers e processamento de fixtures controlados pelo alvo só podem rodar dentro do sandbox aprovado do SO:
sem rede externa, ambiente vazio com allowlist, alvo e ferramentas somente-leitura, escrita restrita ao seu
diretório scratch/ e limites baixos de CPU, memória, processos, tamanho de arquivo, disco e wall-clock.
Loopback isolado é permitido apenas para fixture local. Se faltar qualquer controle, não execute: retorne
needs_validation com esse bloqueador exato. Prefira teste unitário existente, harness mínimo de função,
chamada de serviço com tenants fictícios, pequena fixture malformada, escalonamento determinístico de corrida
ou política renderizada localmente. Não instale nem baixe ferramentas.

Registre entrada exata, comando, limites e resultado mínimo observado. Para o ambiente, registre apenas nomes
de variáveis da allowlist e valores seguros não secretos necessários à reprodução. Nunca capture o ambiente do host,
variáveis herdadas, credenciais, estado de autenticação ou caminhos não relacionados. Após o encerramento do sandbox,
apenas código confiável do coordenador pode promover arquivos pré-declarados de scratch/, seguindo literalmente o
procedimento de promoção incluído neste prompt. Se a promoção falhar para evidência decisiva, retorne needs_validation
com o bloqueador exato de promoção. Nunca estresse disponibilidade, nunca acione alvos vivos e nunca use credenciais reais.

Um fato de deploy, navegador, provedor, broker, SO, proxy, pacote, segredo ou identidade fora do código-fonte
não prova nem segurança nem vulnerabilidade. Se tal fato for decisivo, retorne um registro needs_validation com
a observação exata ausente e um plano seguro de verificação pelo proprietário.
```

#### Procedimento de promoção — copiar literalmente em todo prompt de hunter

```text
Procedimento de promoção de artefatos (exclusivo do código confiável do coordenador):
Apenas para sua referência: o coordenador executa estes passos; você jamais os executa.

Antes da execução, o coordenador abre e retém descritores de diretório confiáveis e não herdáveis para as raízes scratch/ e artifacts/ do agente, registrando uma allowlist de arquivos esperados relativos a scratch/ além de limites explícitos de bytes por arquivo e cumulativos. Nunca passe esses descritores ao subagente ou ao sandbox. Após o término do sandbox e de todos os seus processos, o código confiável do coordenador promove cada arquivo permitido separadamente:

1. Valide o caminho relativo declarado: rejeite componentes absolutos, vazios, `.`, `..` ou com links simbólicos (symlinks).
2. Percorra cada componente pai a partir do descritor retido da raiz scratch/ usando operações relativas a descritor sem seguir links (no-follow); nunca reabra por caminho textual.
3. Abra o arquivo folha com O_NOFOLLOW e O_NONBLOCK (no-follow e não bloqueante).
4. Verifique com fstat que se trata de um arquivo regular com contagem de links exatamente igual a 1 (nlink == 1) e dentro dos limites registrados por arquivo e cumulativos em bytes.
5. Imponha novamente esses limites durante a leitura a partir desse descritor.
6. Copie exatamente o tamanho verificado, repita fstat e rejeite qualquer mudança de identidade, tipo, contagem de links ou tamanho.
7. Para o destino, percorra cada componente pai a partir do descritor retido da raiz artifacts/ com operações relativas sem seguir links; exija que cada componente existente seja um diretório real e crie qualquer diretório ausente de forma exclusiva antes de reabri-lo e verificá-lo no-follow.
8. Crie o arquivo folha de forma exclusiva sem seguir links, verifique que o destino aberto é um arquivo regular com contagem de links igual a 1 e copie a partir do descritor de origem verificado sem reabrir nenhum dos caminhos.
9. Utilize APIs equivalentes imunes a condições de corrida (race-safe) em sistemas não-POSIX.
10. Nunca copie recursivamente nem faça glob de scratch/, nunca extraia arquivos compactados diretamente em artifacts/ e nunca abra ou promova symlinks, FIFOs, sockets, dispositivos, diretórios, hard links, arquivos em mutação ou arquivos que excedam seu limite.
11. Se qualquer verificação estiver indisponível, não puder ser imposta ou falhar, descarte a entrada de scratch/; se ela for evidência decisiva, retenha o candidato como needs_validation registrando o bloqueador exato de promoção.
```

#### Regras centrais de validação (`Candidate gate`) — incluir em todo prompt de hunter

```text
## Candidate gate

1. Todo candidato exige um trace completo relativo ao repositório e evidência da causa-raiz alegada,
   incluindo a análise do controle visível mais forte no caminho.
2. Um candidato proposto como confirmed exige resultado local observado delimitado, impacto real através de
   uma fronteira declarada, condições completas e ausência de camada preventiva visível no código.
3. Não promova um crash genérico a execução de código, processamento comum a negação de serviço compartilhada,
   nem uma ação autorizada do próprio principal a ganho de privilégio.
4. Se um fato necessário não for visível no código-fonte nem observável localmente em sandbox, utilize
   needs_validation. Nomeie os bloqueadores exatos; não atribua severidade nem complete especulativamente.
5. Ausência de boa prática sem principal ou recurso afetado é nota de hardening ou item excluído,
   não um achado. Um candidato refutado pelo próprio código não é needs_validation.
6. Use sempre o mesmo fingerprint derivado do código para a mesma causa-raiz em qualquer estado.
   Ele deve obedecer a `^[A-Za-z0-9][A-Za-z0-9._:/@+-]*$` e não pode conter linha, onda, agente, severidade ou veredito.
7. Retorne um array candidates vazio quando nada sobreviver a estes critérios.
```

## Limites da validação local

A execução local serve exclusivamente para confirmação mínima, nunca para expansão de impacto:

- **Permitido apenas no sandbox obrigatório do SO:** builds offline com dependências já presentes; processos em loopback isolado com estado fictício; testes unitários e de integração; processamento de pequenas fixtures; sanitizers (ASan/UBSan/TSan); testes delimitados de regressão/fuzzing; verificações determinísticas de concorrência; testes locais de navegador/emulador com contas fictícias; manifestos renderizados e avaliação de políticas com identidades simuladas; chamadas externas/pagas mockadas.
- **Proibido:** tráfego contra ambientes vivos ou implantados; requisições a serviços não iniciados especificamente para o check isolado; instalação de pacotes via rede; contas ou credenciais reais; dados de produção; filas, recursos cloud, runners, registries ou serviços de assinatura compartilhados; publicação de pacotes; geração de carga/estresse ou custos; qualquer ação além do resultado mínimo na fronteira com dados fictícios.

## Resultado estruturado do Hunter

Retorne exatamente um único objeto JSON, sem texto ou explicações ao redor:

```json
{
  "units": [
    {
      "coverage_id": "um ID atribuído",
      "disposition": "covered|candidate|blocked",
      "reviewed_paths": ["caminho/relativo/ao/repo"],
      "checks": [
        {
          "agent_id": "proprietário canônico deste check",
          "reviewed_paths": ["caminho/relativo/revisado/neste/check"],
          "invariant": "controle específico verificado nesta unidade",
          "method": "source|local",
          "result": "o que o código ou o teste delimitado comprovou",
          "artifact": "agents/<agent-id>/artifacts/arquivo para local, null para source"
        }
      ],
      "candidate_fingerprints": [],
      "unresolved": []
    }
  ],
  "candidates": [],
  "hardening": ["nota concreta de endurecimento que não constitui vulnerabilidade"],
  "uncovered": [
    {
      "surface": "...",
      "boundary": "...",
      "subsystem": "...",
      "attack_class": "...",
      "starting_paths": ["caminho/relativo/ao/repo"],
      "reason": "por que isto exige sua própria unidade determinística de cobertura"
    }
  ]
}
```

Cada item de `candidates` segue o formato de `report-schema.json`, usando `proposed_verdict` no lugar de `verdict`:

- `proposed_verdict: "confirmed"`: inclua todos os campos exigidos pelo ramo `confirmed` de `report-schema.json` exceto `verdict`: `fingerprint`, `title`, `description`, `root_cause`, `intended_behavior`, `trace` ordenado (`entrypoint` → `propagation` → `sink`), `evidence`, `conditions`, `execution` neutra em relação ao alvo, `remediation`, `severity` e `confidence`. `overall_severity` jamais pode superar o impacto demonstrado (`impact.score`).
- `proposed_verdict: "needs_validation"`: inclua todos os campos exigidos pelo ramo `needs_validation` exceto `verdict`: `fingerprint`, `title`, `description`, `claimed_root_cause`, `trace`, `evidence`, `blockers` não vazio e `validation_plan` com ao menos um passo aplicável em `local` ou `deployment`. Não inclua `severity`, `execution`, `remediation`, `reason` ou `root_cause`.

## Consolidação pelo coordenador e atualização do ledger

O coordenador valida cada resultado de unidade, mapeia para exatamente um `coverage_id` atribuído e atualiza somente aquela unidade no ledger. Rejeite IDs ausentes ou duplicados, `agent_id` inseguros, checks `source` com artefato não nulo e checks `local` cujos artefatos não tenham sido promovidos pelo código confiável do coordenador para `agents/<check.agent_id>/artifacts/`. Copie `reviewed_paths`, `checks` (para `local_checks`), `candidate_fingerprints` e `unresolved` no ledger, mantendo as notas de `hardening` em campo auxiliar para a Fase 6. Uma conclusão malformada ou sem suporte mantém a unidade como `planned` para reatribuição. Execute `validate-coverage-ledger.cjs` após cada atualização; um ledger inválido jamais pode guiar novas atribuições.

Consolide candidatos por `fingerprint` e depois por causa-raiz. Uma única causa-raiz exposta em múltiplos endpoints constitui um único candidato com o trace mais forte e completo. Controles ausentes independentes usam fingerprints separados.

## Ondas de críticos de cobertura (`Coverage-critic waves`)

Imediatamente após cada onda de *hunters*, utilize a invocação reservada em um subagente `research` fresco atuando como crítico de cobertura pós-onda. Ele recebe `architecture.md`, o `coverage-ledger.json` completo, os fingerprints e estados atuais dos candidatos e o resumo de lacunas anteriores. Ele lê o código-fonte, mas não escreve arquivos nem executa código. Exija exatamente este JSON:

```json
{
  "missing_units": [
    {
      "surface": "...",
      "boundary": "...",
      "subsystem": "...",
      "attack_class": "...",
      "starting_paths": ["caminho/relativo/ao/repo"],
      "selected_companion_blocks": ["FILE.md#section"],
      "excluded_blocks": [{"block": "FILE.md#section", "reason": "..."}],
      "reason": "lacuna de cobertura fundamentada no código-fonte"
    }
  ],
  "reassign_ids": ["id-existente-que-nao-fechou-adequadamente"],
  "resolved_prior_leads": ["fingerprint"],
  "stop": false
}
```

O crítico procura pontos de entrada não mapeados, caminhos paralelos ignorados, modos de ciclo de vida ausentes, classes selecionadas sem unidade correspondente, exclusões injustificadas, unidades fechadas sem evidência proporcional e lacunas anteriores não endereçadas. Ele propõe cobertura, nunca achados.

Nos perfis `standard` e `deep`, quando o crítico pós-onda não reportar `missing_units` aceitas nem `reassign_ids` legítimos e não restarem unidades `planned`, utilize a reserva separada para um **crítico final limpo (`final-clean critic`)** distinto. Declare cobertura limpa somente quando esse segundo crítico independente também retornar zero pendências. No perfil `quick`, executa-se exatamente uma onda de *hunters* seguida de um único crítico final, registrando descobertas e reatribuições aceitas como `deferred` com razão `quick_profile_final_critic`.
