# Caça em IA, LLMs e Agentes Autônomos (`AI, LLM, and Agent Hunting`)

#### When to use this file

Utilize este guia sempre que um modelo de linguagem participar de uma decisão sensível à confiança: chatbots e assistentes, pipelines de RAG, memória persistente de agentes, laços de *tool calling*, servidores e clientes MCP (*Model Context Protocol*), código que constrói prompts a partir de entradas não confiáveis ou código que consome saídas do modelo para executar ações. O fluxo crítico de dados é *conteúdo não confiável → modelo ou memória → capacidade, autoridade ou sink*.

Use este arquivo em conjunto com `ATTACK-CLASSES.md`, nunca em substituição a ele. Transporte, controle de acesso, construção de queries, uso de filesystem e renderização continuam sendo fronteiras clássicas de confiança (veja também controles `AI01–AI10`, `INT01–INT03` e `STK04`). Divida alvos grandes em recuperação (RAG), memória, despacho de ferramentas, MCP e tratamento de saída.

## Core discipline (incluir em todo prompt de agente para este domínio)

```
- Prompt injection isoladamente não é um achado confirmado. Exija uma falha determinística de fronteira no código: conteúdo alcança o contexto de outro principal, invoca autoridade que o solicitante não possui, revela dados aos quais ele não tem leitura ou aciona um sink que ele não alcançaria diretamente.
- Saídas do modelo, memórias, descrições de ferramentas e respostas MCP são entradas não confiáveis. Aponte exatamente o código que concede autoridade, confia na saída, grava estado durável ou alimenta um sink.
- Um prompt de guardrail não é fronteira de segurança. Contabilize apenas verificações determinísticas em código, autorização escopada por recurso, isolamento, vinculação de intenção (action-binding) e credenciais restritas.
- Declare o atacante, principal afetado, identidade efetiva de execução, recurso, ação exata, autoridade utilizada e impacto observável. Uma solicitação direta intencional do usuário para usar sua própria autoridade legítima não é defeito de delegação apenas por passar por um modelo.
- Autorização e vinculação de ação (action binding) são controles distintos. Conteúdo controlado por atacante que dispara uma ação sob a autoridade válida da vítima constitui falha de action-binding quando a vítima não solicitou nem aprovou intencionalmente aquela ação exata.
- Classifique candidatos como `confirmed` somente após evidência no código e validação local delimitada comprovarem fronteira e resultado. Use `needs_validation` quando o comportamento do provedor, deploy, modelo, renderizador ou identidade não for observável localmente.
```

## Context, retrieval, and memory attack classes (subagent_type: `general`)

**Indirect injection through retrieved or ingested content**
Um atacante consegue gravar um documento RAG, página indexada, arquivo, e-mail, issue, resposta de ferramenta ou metadado que ingressa no contexto de modelo de outro principal. Rastreie quem pode escrever em cada fonte, como a busca filtra o escopo, qual sessão consome o conteúdo e quais ferramentas estão habilitadas nela. Avalie separadamente isolamento, autorização sobre recursos e vinculação à intenção do usuário consumidor.

**Cross-session or cross-tenant context bleed**
Histórico de conversas, embeddings, resultados de busca vetorial ou caches semânticos de prompts indexados com chaves amplas demais. Verifique filtros de `tenant_id` e ACL na própria query vetorial/relacional e em toda chave de cache. Um campo `tenant_id` salvo no objeto não garante isolamento se uma rota alternativa, cache compartilhado ou processamento em lote omitir o filtro.

**Persistent memory poisoning**
Conteúdo controlado pelo atacante ou resumos gerados pelo modelo são gravados em memória persistente que moldará tarefas futuras, outros usuários ou sessões privilegiadas. Revise quem pode criar, atualizar, mesclar e excluir memórias; sua proveniência e escopo de tenant; se observações não confiáveis viram instruções duráveis; e se a leitura distingue preferências de usuário de políticas de ferramentas.

**Prompt role and provenance confusion**
A montagem do prompt permite que texto não confiável se passe por mensagem `system`, turno anterior, resultado de ferramenta, política ou registro de memória confiável. Procure concatenação manual de strings, histórico sem tipagem estrita, campos `role` controlados pelo cliente e round-trips de serialização que apagam rótulos de origem.

## Tool and action attack classes (subagent_type: `general`)

**Tool-argument injection into a downstream sink**
Argumentos produzidos pelo modelo chegam a SQL, shell, sistema de arquivos, fetch de URLs (SSRF) ou APIs privilegiadas sem validação no handler. Trate o JSON Schema da ferramenta apenas como parsing sintático: siga cada campo decodificado até o sink final.

**Excessive agency and confused-deputy authority**
O agente opera com identidade de serviço ampla (*service_role*) enquanto o handler da ferramenta não revalida a permissão do usuário solicitante sobre o recurso alvo. Verifique a identidade efetiva e se o chamador conseguiria realizar aquela exata operação pela API convencional.

**Action-confirmation and approval binding**
Um usuário aprova uma ação descrita (HITL), mas a execução consegue usar argumentos alterados, outro recurso, outro principal ou um turno posterior do modelo — ou conteúdo externo dispara efeitos colaterais sob a autoridade válida da vítima sem aprovação intencional para aquela operação exata. Verifique se a confirmação vincula criptograficamente o nome normalizado da ferramenta, o objeto completo de argumentos, o solicitante, o alvo, o valor financeiro, a expiração e a idempotência em retentativas ou sessões retomadas.

**Tool-schema and dispatcher disagreement**
O schema aceita aliases, campos extras, chaves duplicadas, coerções de tipo, objetos aninhados livres ou valores fora de faixa interpretados de modo divergente pelo dispatcher ou handler final. Compare validação de schema, canonicalização e valores default do handler.

**Unbounded delegated action loops**
Uma única requisição enfileira gastos repetidos, envios externos, mutações ou chamadas recursivas de subagentes sem teto de passos/custo por requisição, autorização por ação, propagação de cancelamento ou controle de idempotência (*Denial of Wallet* — `AI06`). Valide por inspeção contábil e loop local estritamente limitado, nunca exaurindo serviços reais.

## MCP and sub-agent trust classes (subagent_type: `general`)

**Sub-agent and MCP trust inheritance**
Uma tarefa delegada herda a sessão completa, credenciais mestras, memória irrestrita ou todas as ferramentas em vez da autoridade mínima necessária. Verifique estreitamento de capacidades (*capability narrowing*), audiência de credenciais e se o retorno do subagente é tratado como entrada não confiável.

**MCP server and tool identity confusion**
Chamadas ou respostas roteadas por nomes de servidores, nomes de ferramentas, `request_id`, URIs de recursos ou aliases influenciáveis pelo atacante/modelo em vez da conexão autenticada e da requisição pendente correlacionada. Verifique colisão de namespaces entre servidores MCP e rebinding após reconexão.

**MCP metadata and schema as policy**
Descrições de ferramentas, anotações (`readOnlyHint`, `destructiveHint`), prompts ou schemas fornecidos por um servidor MCP externo tratados como política de segurança autoritativa. Esses metadados orientam o modelo, mas jamais concedem permissão: localize a allowlist determinística e a autorização no handler.

## Output and disclosure attack classes (subagent_type: `general`)

**Insecure output rendering**
Saída do modelo atinge um sink executável de HTML, Markdown, template, URL (`javascript:`/`data:`) ou comando sem sanitização e CSP adequadas (`AI10` / `APP07`).

**Sensitive context extraction**
O contexto montado contém segredos, dados de outro tenant, código privado ou chaves que concedem acesso, e a resposta influenciada pelo usuário os expõe (`AI02` / `AI08`). Divulgação de instruções genéricas de estilo que não cruzam fronteira de dados não é vulnerabilidade.

## Universal moves (aplicar transversalmente neste domínio)

- Desenhe quatro mapas primeiro: cada identidade de execução, cada capacidade/ferramenta, cada fonte gravável de contexto/memória e cada destino de saída. Conecte o principal inicial à autoridade final.
- Comece pelas ferramentas com efeitos colaterais (*side-effecting tools*) e caminhe de trás para frente passando por handler, dispatcher, schema, aprovação HITL, contexto do modelo, busca RAG e ingestão.
- Compare caminhos diretos, enfileirados, retentativas, retomada de sessão, lote e subagentes para a mesma ação: a trava mais forte deve atuar após os argumentos finais estarem imutáveis e imediatamente antes do efeito colateral.

## Validation rules (aplicar antes de reportar QUALQUER achado aqui)

1. Nomeie a fronteira cruzada e o resultado observável: atacante, principal ou recurso compartilhado afetado, identidade de execução, alvo e ação/exposição não autorizada ou não solicitada.
2. Em alegações de *confused deputy*, prove que falta autorização solicitante-recurso no handler e que o atacante não faria o mesmo pela interface comum. Em falhas de *action-binding*, prove que conteúdo controlado pelo atacante causou efeito sob a autoridade da vítima sem solicitação ou aprovação intencional para aquela ação exata.
3. Em alegações de memória ou RAG, cite tanto a escrita controlada pelo atacante quanto a leitura cross-principal ou decisão privilegiada posterior.
4. Em *action-binding* ou divergência de schema, compare o objeto normalizado aprovado/validado com o objeto efetivamente consumido pelo handler em teste local delimitado.
5. Em identidade MCP, verifique conexão autenticada, correlação de requisição, namespace de ferramentas e credencial efetiva; marque `needs_validation` quando depender de roteamento externo.
6. Retorne `confirmed` somente com trace completo no código e resultado concreto demonstrado; retorne `needs_validation` para fatos pendentes externos especificando o check seguro correspondente.
