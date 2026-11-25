# Validação, Saída Estruturada, Verificação e Relatórios

### Fase 3: Validar independentemente cada candidato (Validação Adversarial)

Após a passagem limpa do crítico de cobertura (ou parada antecipada explicitamente registrada), consolide os candidatos da Fase 2 e as confirmações anteriores de mesma origem por `fingerprint` estável e causa-raiz. Entregue cada candidato único proposto (`confirmed` ou `needs_validation`) a um subagente verificador `general` fresco **que não participou da sua caça**. Um verificador pode ler artefatos anteriores, mas deve reler cada localização citada no código-fonte atual e reproduzir de forma independente qualquer teste local decisivo dentro do sandbox seguro.

Atribua a cada verificador um ID canônico exclusivo em minúsculas com `<output-dir>/agents/<verifier-id>/scratch/` e `artifacts/` de propriedade do coordenador. O verificador escreve apenas em `scratch/`, recebe somente o candidato sob análise, os checks e artefatos vinculados, os fatos arquiteturais necessários, os blocos de validação dos domínios pertinentes, o procedimento de promoção abaixo e os ramos `confirmed`, `needs_validation` e `rejected` de `report-schema.json`. Ele jamais recebe a conclusão de outro verificador.

#### Prompt do verificador de candidato (`Candidate-verifier prompt`)

```text
Você não escreveu este candidato. Tente refutá-lo a partir do código-fonte do repositório e de evidências
locais delimitadas. Não contate endpoints implantados nem serviços externos/compartilhados. Execute
código controlado pelo alvo exclusivamente dentro do sandbox aprovado do SO: sem rede externa,
ambiente vazio com allowlist, alvo e ferramentas somente-leitura, escrita restrita a scratch/ e
limites explícitos e baixos de recursos e wall-clock. Se qualquer controle estiver indisponível,
não execute; mantenha a capacidade ausente exata como bloqueador em needs_validation.
Trate toda entrada em scratch/ como controlada pelo alvo após a execução. Somente código confiável do
coordenador pode promover arquivos pré-declarados de scratch/, seguindo literalmente o bloco de promoção
incluído neste prompt. Se a promoção falhar, não use o arquivo como evidência.

1. Verifique cada arquivo de trace e evidence, número de linha positivo, escopo e descrição.
   Confirme que o primeiro item é um ponto de entrada real de menor confiança (entrypoint) e o último é
   o sink ou efeito de fronteira alegado.
2. Reconstrua os controles mais fortes visíveis no código para validação, identidade, autorização,
   normalização, ciclo de vida, framework e contenção ao longo do caminho.
   Onde o resumo de arquitetura citar um software comparável, observe se ele compartilha o padrão — apenas
   para calibração, nunca como pretexto para descartar uma falha demonstrada.
3. Para um candidato proposto como confirmed, reproduza independentemente o resultado mínimo observado
   sempre que possível. Verifique entradas, formato da interface, condições e principal/recurso fictício afetado.
   Não infira resultado mais forte nem prossiga após comprová-lo.
4. Verifique que likelihood, impact, overall_severity (nunca superior ao impact), confidence e a correção
   proposta correspondem estritamente ao que a evidência comprova.
5. Para um candidato proposto como needs_validation, decida se o bloqueador está genuinamente fora da
   observação estática/local. Se o código-fonte refutar o trace, rejeite-o (rejected). Se o fato ausente
   continuar decisivo, mantenha needs_validation tornando os planos local e deployment exatos e não destrutivos.
6. Preserve o fingerprint para a mesma causa-raiz derivada do código em todos os estados.

Retorne exatamente um objeto JSON e nenhum texto ao redor:
{"decision": "confirmed|needs_validation|rejected", "record": { ... }}
onde record obedece integralmente ao ramo correspondente do schema incluído neste prompt.
```

Copie literalmente este bloco de promoção em todo prompt de verificador de candidato:

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

Um verificador só pode promover `needs_validation` para `confirmed` após comprovar independentemente o caminho completo e o resultado local delimitado. Rebaixe uma proposta `confirmed` para `needs_validation` quando um fato específico de deploy ou runtime permanecer desconhecido. Use `rejected` quando o código-fonte, o comportamento local, um controle visível, a ausência de impacto real ou um pré-requisito impossível refutar a alegação. `needs_validation` jamais serve de depósito para ideias especulativas.

### Fase 4: Gravar e validar `findings.json`

O coordenador grava todos os registros decididos de forma independente em `<output-dir>/findings.json`, ordenados lexicograficamente por `fingerprint`:

- **`confirmed`**: vulnerabilidades fundamentadas no código com evidência completa de execução local, condições, remediação específica, severidade (`likelihood`, `impact`, `overall_severity`) e `confidence`. Não pode conter `claimed_root_cause`, `blockers`, `validation_plan` nem `reason`.
- **`needs_validation`**: candidatos fundamentados no código com bloqueador exato em `blockers` e ao menos um plano aplicável em `validation_plan.local` ou `validation_plan.deployment`. Não pode conter `severity`, `execution`, `remediation`, `reason` nem `root_cause`.
- **`rejected`**: candidatos refutados durante a validação com `claimed_root_cause` e `reason`, preservados para evitar repetição em auditorias futuras sem fatos novos. Não pode conter `severity`, `execution`, `remediation`, `blockers`, `validation_plan` nem `root_cause`.

Leia `report-schema.json` imediatamente antes de gravar (`additionalProperties: false` é aplicado em todos os níveis). Um `trace` com múltiplos passos começa obrigatoriamente em `entrypoint`, termina em `sink` e usa `propagation` apenas nos passos intermediários; traces de uma única entrada usam `entrypoint` ou `sink`. `overall_severity` jamais pode exceder `impact.score`.

Execute obrigatoriamente os dois validadores:

```sh
node <skill-dir>/validate-findings.cjs <output-dir>/findings.json
node <skill-dir>/validate-coverage-ledger.cjs <output-dir>/coverage-ledger.json
```

Corrija todo erro estrutural ou semântico antes de prosseguir para a Fase 5.

### Fase 5: Verificar os registros finais com olhos frescos

Inicie um subagente `research` fresco em paralelo para cada registro final `confirmed` e `needs_validation`. Este verificador audita o registro estruturado final (não o rascunho do *hunter*), permanecendo estritamente dentro dos limites de código-fonte/local. (No perfil `quick`, as Fases 3 e 5 fundem-se em um único verificador independente por candidato; nos perfis `standard` e `deep`, as duas passagens permanecem separadas).

Para registros `confirmed`, exija a conferência de:

1. Todos os caminhos relativos em `trace` e `evidence`, números de linha exatos, escopo e operação descrita.
2. Interface real de entrada e formato exato do input local.
3. Todas as condições, etapas de parser/política, ausência de camadas preventivas visíveis e resultado local observado.
4. Principal/recurso afetado e impacto demonstrado.
5. Calibração de severidade: probabilidade realista, impacto demonstrado e `overall_severity <= impact.score`.
6. Estratégia de remediação e `code_changes`, verificando se a correção impõe o invariante na fronteira autoritativa em vez de apenas deslocar confiança.

Para registros `needs_validation`, exija a conferência de:

1. O caminho de código é real e sustenta estritamente o `claimed_root_cause` declarado.
2. Cada bloqueador listado é decisivo e realmente insolúvel apenas com inspeção local.
3. O candidato nomeia uma fronteira clara e um resultado concreto possível, não uma preocupação abstrata.
4. Ao menos um campo de `validation_plan` está presente, preciso e seguro (`deployment` pede observação de configuração/política pelo proprietário, nunca tráfego de ataque contra produção).
5. O `fingerprint` é consistente com registros anteriores/atuais da mesma causa-raiz.

Cada verificador retorna exatamente um objeto JSON: `{"decision":"verified","fingerprint":"..."}` ou `{"decision":"replace","reason":"...","record":{...}}`. Se uma substituição (`replace`) na Fase 5 promover um registro para veredito mais forte (incluindo qualquer promoção a `confirmed`) ou alterar materialmente causa-raiz, trace, payload/resultado observado, impacto ou severidade, submeta essa substituição completa a **outro verificador independente inédito** antes de aceitá-la, reexecutando ambos os validadores `.cjs` após qualquer aplicação. Defina `run_status: "complete"` somente quando todo candidato do ledger tiver disposição final independente e todos os registros retidos passarem na Fase 5.

### Fase 6: Produzir relatórios neutros derivados dos registros finais

Somente após a aprovação integral na Fase 5, derive os relatórios em Markdown a partir dos registros finais verificados, do ledger e das notas de `hardening`. Os arquivos em prosa jamais alteram veredito, severidade, bloqueador ou impacto demonstrado em `findings.json`.

#### `REPORT.md`

Inclua:

1. Perfil da execução (`quick`/`standard`/`deep`), escopo, orçamento (agentes consumidos vs planejados), `source_ref`, declaração de execução restrita a código-fonte e sandbox local, uso de execuções anteriores e cobertura explicitamente `deferred` ou `out_of_scope`.
2. Resumo executivo objetivo da postura de segurança encontrada.
3. Tabela de achados confirmados (`confirmed`): severidade, título, fronteira afetada e resultado observado em uma linha.
4. Detalhamento conciso de cada achado confirmado: localização no código, principal de menor confiança, reprodução nativa delimitada, condições, resultado observado, impacto, justificativa de prioridade e menor correção eficaz (referenciando controles `APP`/`AI`/`DB`/`INF`/`STK`/`PRO`/`CLI`/`IPC`/`MEM` quando aplicável).
5. Tabela separada `NEEDS VALIDATION`: título, trace no repositório, bloqueador exato, próximo passo local delimitado e verificação segura de implantação pelo proprietário — **sem severidade** e sem chamá-los de vulnerabilidades confirmadas.
6. Notas separadas de hardening (defesa em profundidade) e padrões positivos encontrados no código.
7. Resumo quantitativo de cobertura extraído do `coverage-ledger.json` (contagens `covered`, `candidate`, `blocked`, `deferred`, `out_of_scope`, exclusões relevantes e resultado do crítico final).

#### `FINDINGS-DETAIL.md`

Para cada achado confirmado `medium`, `high` ou `critical`, documente o caminho completo no código e a reprodução local neutra na interface nativa do alvo (chamada HTTP, função de biblioteca, fixture de parser, CLI, IPC ou manifesto renderizado):

- `trace` ordenado e `evidence` relativos ao repositório;
- principal/atacante fictício e recurso fictício afetado;
- input nativo, invocação ou fixture e instruções exatas delimitadas;
- saída observada e o invariante de segurança violado;
- condições e contenção existente;
- remediação em nível de código-fonte e caso de teste de regressão.

#### `NEEDS-VALIDATION.md`

Para cada registro pendente de validação, copie o trace verificado, evidências, bloqueador exato, fronteira afetada e plano seguro de resolução local ou de implantação pelo proprietário, mantendo-os como pistas priorizadas sem severidade.

Mantenha o relatório estritamente proporcional às evidências. Uma auditoria limpa pode legitimamente terminar com **zero** achados confirmados: declare esse resultado e os limites remanescentes de cobertura/validação sem inventar achados `low` artificiais.
