# Caça em Exaustão de Recursos e Disponibilidade (`Resource Exhaustion and Availability Hunting`)

#### When to use this file

Utilize este guia quando requisições, mensagens, arquivos, estados de tenant ou tarefas de agentes não confiáveis puderem consumir CPU, memória, disco, pool de conexões, slots de workers, cotas pagas de APIs/LLMs ou capacidade de filas, ou puderem travar (*deadlock*) / derrubar (*crash*) um serviço compartilhado (veja também controles `APP12`, `AI06` e `STK03`). Este domínio separa vulnerabilidades reais de disponibilidade auditáveis no código de meros gargalos genéricos de performance. **Jamais valide estressando serviços vivos ou compartilhados.**

## Core discipline (incluir em todo prompt de agente para este domínio)

```
- Exija um caminho claro entrada-para-custo (input-to-cost), ausência de limite efetivo e impacto sobre outro usuário, serviço compartilhado, função crítica de segurança ou orçamento financeiro do operador. Lentidão restrita apenas ao próprio processo local do solicitante não é vulnerabilidade de serviço.
- Ausência isolada de rate limit não basta: verifique limites de tamanho de body/mensagem/arquivo, concorrência, filas, timeouts, restrições de banco, gateways upstream e cotas por tenant antes de declarar um caminho ilimitado.
- Jamais execute testes de estresse, carga, saturação ou produção. Use análise assintótica, pequenas fixtures de fronteira, mocks de chamadas pagas, limites estritos de recursos locais e testes determinísticos de cancelamento.
- Declare o custo para o atacante versus o trabalho gerado no servidor, persistência, escopo de impacto e capacidade de recuperação. Uma única requisição pequena com efeito superlinear ou persistente difere materialmente de ataque volumétrico bruto.
- Use `confirmed` para falhas de limite visíveis no código demonstradas com segurança local. Use `needs_validation` quando limites de WAF/gateway, topologia de deploy, autoscaling ou cotas externas estiverem fora do repositório.
```

## Computational amplification attack classes (subagent_type: `general`)

**Superlinear parsing, matching, or evaluation**
Pequenas entradas aceitas disparam *catastrophic backtracking* em expressões regulares (ReDoS), parsing profundamente aninhado (JSON/XML/GraphQL/AST), validação recursiva, expansão de templates ou colisões algorítmicas. Derive a complexidade aceita e demonstre localmente a curva de crescimento com entradas pequenas e controladas.

**Decompression and representation amplification**
Arquivos compactados (*zip bombs*, gzip/brotli/zstd), imagens esparsas, XML/YAML com entidades/aliases ou tabelas de compressão de protocolo que se expandem ordens de grandeza além do tamanho verificado no upload/transferência sem teto de bytes descompactados.

**Database and downstream query amplification**
Uma requisição pequena dispara full table scans patológicos, joins cartesianos, consultas GraphQL com dezenas de aliases (`APP17`), paginação sem limite superior (`limit=9999999`) ou efeito *fan-out* de centenas de chamadas downstream.

## Resource accumulation attack classes (subagent_type: `general`)

**Unbounded buffering and cardinality**
Corpos de streaming, uploads em memória, sessões não autenticadas, chaves únicas de cache, *labels* de métricas Prometheus de alta cardinalidade controladas pelo usuário ou jobs pendentes acumulando-se sem teto por item e agregado.

**File descriptor, handle, and temporary-resource leaks**
Fluxos malformados, erros precoces ou conexões abortadas que pulam blocos `finally`/`defer` e vazam sockets, descritores de arquivo, cursores/conexões do pool de banco, timers, goroutines (`STK03`) ou arquivos temporários em disco.

**Detached work after cancellation**
Desconexão HTTP do cliente, timeout ou falha de autorização retorna resposta rápida, mas deixa queries pesadas no banco, inferências caras de LLM (`AI06`) ou workers em background rodando sem propagação de `AbortSignal` / `context.Context`.

## Quota and scheduling attack classes (subagent_type: `general`)

**Pre-authentication work imbalance**
Parsing pesado, hashing criptográfico custoso (ex.: Argon2id em senhas de 1 MB sem truncamento/limite prévio de tamanho!), descompressão ou chamadas externas executados *antes* da autenticação e do limite inicial de tamanho/taxa.

**Quota-accounting scope and reset gaps**
Contabilização de rate limit ou orçamento de tokens (`AI06`) baseada em cabeçalho `X-Forwarded-For` falsificável pelo cliente, chave de rota incorreta, corridas distribuídas (*Check-Then-Act* na cota) ou overflow numérico que permite escapar do orçamento do tenant.

**Worker, pool, and priority starvation**
Tarefas lentas ou bloqueantes controladas por um único tenant seguram locks globais, esgotam todas as conexões do PgBouncer/pool de banco ou travam o Event Loop do Node.js, paralisando tenants não relacionados (*Noisy Neighbor*).

## Failure and recovery attack classes (subagent_type: `general`)

**Reachable fatal error or deadlock**
Uma entrada não confiável alcança `panic` não recuperado, `unwrap()`/`assert()` fatal, `uncaughtException`, saída abrupta do processo ou ciclo de deadlock em um processo compartilhado entre múltiplos usuários.

**Retry storm and fail-open amplification**
Timeouts ou falhas parciais acionam retentativas imediatas e ilimitadas sem *exponential backoff*, *jitter*, *circuit breaker* ou deduplicação, multiplicando exponencialmente a carga interna.

**Poison-record and head-of-line blocking**
Uma única mensagem malformada (*poison pill*) falha repetidamente na frente de uma fila ou partição compartilhada sem política de quarentena/DLQ após $N$ tentativas, travando o processamento de todos os demais clientes.

**Unsafe recovery and capacity rollback**
Rotinas de reinicialização ou restauração que recarregam automaticamente na memória o mesmo estado acumulador ou o mesmo registro venenoso, derrubando o serviço em loop contínuo (*crash loop*).

## Universal moves (aplicar transversalmente neste domínio)

- Construa uma tabela `entrada → recurso`: tamanho máximo aceito na borda, trabalho executado antes do auth, fan-out downstream, persistência, pool compartilhado, responsável pelo limite/limpeza e recuperação.
- Compare limites por item com limites agregados: dez mil itens válidos de 1 byte podem contornar um limite por mensagem e exaurir a memória global ou a cota do provedor.
- Valide exclusivamente em fixture isolada com limites rígidos de CPU/memória/tempo e pequenos pontos de medição, mockando qualquer API paga.

## Validation rules (aplicar antes de reportar QUALQUER achado aqui)

1. Nomeie a entrada não confiável, o esforço do atacante, a amplificação ou retenção no servidor, o raio de impacto compartilhado e o comportamento de recuperação. Ausência teórica de limite sem impacto compartilhado concreto é nota de hardening.
2. Confirme que nenhuma camada visível no repositório (parser, framework, fila ou cota) já bloqueia o caminho; limites de infraestrutura externa desconhecidos exigem `needs_validation`.
3. Estabeleça matematicamente/localmente a curva superlinear, o vazamento repetível após fechamento ou o alcance fatal no processo compartilhado.
4. Priorize por baixo custo para o atacante, acesso pré-autenticação, impacto cross-tenant e baixa capacidade de auto-recuperação — nunca valide derrubando serviços reais.
5. Retorne `confirmed` apenas com prova local segura e impacto compartilhado real; retorne `needs_validation` detalhando o limite de gateway, topologia ou cota que o proprietário deve checar.
