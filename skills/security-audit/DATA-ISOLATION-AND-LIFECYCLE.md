# Caça em Isolamento e Ciclo de Vida de Dados (`Data Isolation and Lifecycle Hunting`)

#### When to use this file

Utilize este guia quando o alvo armazenar dados multi-tenant ou sujeitos a controle de acesso, derivar cópias em índices de busca, caches, vetores (RAG), analytics ou logs, emitir URLs assinadas de objetos, exportar/importar registros, migrar schemas ou prometer garantias de exclusão (LGPD/GDPR), revogação e retenção. Este domínio acompanha um item de dado através de todas as suas cópias e transições de estado (veja também controles `APP03`, `DB01–DB08`, `INT03` e `GOV01`).

## Core discipline (incluir em todo prompt de agente para este domínio)

```
- Ter uma coluna `tenant_id` ou `owner_id` na tabela não é isolamento. Localize a cláusula WHERE, chave de cache, prefixo de path, política RLS ou filtro vetorial que efetivamente a impõe em CADA caminho de leitura e escrita.
- Rastreie todas as cópias derivadas: dados primários bem protegidos frequentemente vazam em índices de busca (Elasticsearch/Algolia/pgvector), caches (Redis/CDN), analytics, exportações CSV/PDF, previews, logs, réplicas e backups com regras de ACL distintas.
- Exclusão e revogação são contratos de ciclo de vida: verifique cópias atuais, históricas, cacheadas, indexadas, vetorizadas, exportadas, restauradas e enfileiradas dentro da fronteira prometida pelo produto.
- Preferências genéricas de retenção não são automaticamente vulnerabilidades de segurança: exija uma fronteira explícita de acesso a dados ou garantia de exclusão/revogação violada por um leitor não autorizado ou operação posterior.
- Use `confirmed` para linhagem completa visível no código comprovada com tenants fictícios locais. Use `needs_validation` quando políticas externas de bucket, CDN, lag de réplica ou acesso a backups estiverem fora do repositório.
```

## Tenant and object-isolation attack classes (subagent_type: `general`)

**Missing tenant or owner enforcement**
Consultas de leitura, atualização, exclusão, listagem, contagem ou operações em lote que buscam um objeto apenas por ID sem vinculá-lo ao `tenant_id`/`user_id` autenticado na sessão, ou que confiam no `tenant_id` enviado no corpo da requisição (`APP03`). Compare rotas diretas, relacionamentos aninhados (GraphQL/ORM `include`), jobs em background, importadores e endpoints legados.

**Composite-key and namespace collision**
Chaves de cache Redis, caminhos de storage S3, índices de unicidade, IDs de documentos de busca, caches semânticos de IA (`INT03`), arquivos temporários ou chaves de idempotência que omitem o `tenant_id` ou separador seguro (ex.: `tenant + id` onde `"ab" + "c"` colide com `"a" + "bc"`), permitindo colisão ou leitura cruzada entre tenants.

**Policy and query disagreement**
Políticas PostgreSQL RLS (`DB02`), escopos padrão de ORMs, filtros de middleware e clientes administrativos/*raw* (`prisma.$queryRaw`, `unscoped`, chaves `service_role`, conexões em pool sem `DISCARD ALL` — `DB04`) aplicando predicados divergentes ou omitindo `FORCE ROW LEVEL SECURITY`.

**Blob and signed-reference overreach**
Chaves de objetos, IDs de anexos, links compartilháveis ou *presigned URLs* que permitem leitura/escrita fora do namespace do tenant emissor ou continuam válidos indefinidamente após revogação da permissão no banco principal.

## Derived-data and disclosure attack classes (subagent_type: `general`)

**Search, cache, and index ACL drift**
A permissão ou visibilidade de um registro primário é restringida, mas sua cópia em cache, índice full-text, banco vetorial de embeddings (`AI09` / `DB07`), miniatura, feed ou preview não é invalidada nem revalidada no momento da consulta.

**Analytics, logs, traces, and diagnostics as alternate readers**
Conteúdo privado, prompts confidenciais, PII ou tokens emitidos em logs, traces distribuídos, relatórios de erro ou tabelas analíticas acessíveis a perfis de menor privilégio ou misturados entre tenants (`APP19` / `AI02`).

**Enumeration and aggregate oracles**
Contagens, autocompletes, filtros de facetas, ordenações, erros de constraint `UNIQUE` global ou diferenças determinísticas de resposta que revelam a existência ou valores de dados privados de outros tenants.

## Export, backup, restore, and migration attack classes (subagent_type: `general`)

**Export and backup scope expansion**
Rotinas de exportação, relatórios, pacotes de portabilidade LGPD ou snapshots que incluem registros de outros tenants, colunas internas restritas, itens excluídos ou histórico acima do nível de acesso do solicitante, ou cujo artefato final gerado pode ser baixado sem revalidação de autoria.

**Import and restore authority expansion**
Importação de arquivos/backups que contorna validação de proprietário, schema ou ACL, sobrescreve IDs existentes de outros tenants ou recria privilégios administrativos a partir do pacote enviado pelo usuário.

**Migration default and ownership confusion**
Scripts de migração/backfill onde registros antigos ficam com `tenant_id = NULL` (tratado erroneamente como "público/global") ou janelas de *dual-read/dual-write* que aplicam defaults permissivos.

**Backup and replication boundary drift**
Snapshots de suporte, ambientes de homologação populados sem mascaramento estático (`GOV01`) ou réplicas com chaves e controles mais amplos que o banco primário.

## Deletion, revocation, and lifecycle attack classes (subagent_type: `general`)

**Soft-delete and tombstone bypass**
Buscas diretas por ID, joins de relacionamento, buscas vetoriais, links diretos de anexos ou jobs assíncronos que esquecem o predicado `deleted_at IS NULL` / status de revogação e continuam retornando ou operando sobre registros excluídos.

**Stale authorization and derived copy use**
Remoção de membro da organização, revogação de compartilhamento, retirada de consentimento ou rebaixamento de papel que não invalida sessões ativas, tokens de refresh, assinaturas WebSocket/SSE, caches ou jobs agendados pelo ex-membro.

**Retention and queued-work overrun**
A exclusão de titular (`GOV01`) apaga a linha na tabela primária, mas workers assíncronos em fila, retentativas, índices de embeddings ou exportações pendentes recriam ou mantêm os dados além da garantia declarada.

**Restore reintroduces invalid state**
Operações de *undo*, restauração de lixeira ou recuperação de ponto no tempo que restauram membros banidos, credenciais revogadas ou permissões já extintas sem revalidar contra a política vigente.

## Universal moves (aplicar transversalmente neste domínio)

- Escolha um registro confidencial e desenhe todas as arestas: escrita primária, leitura, cache, índice de busca/vetor, evento em fila, exportação, backup, exclusão e restauração, marcando principal e `tenant_id` em cada aresta.
- Compare dois tenants fictícios locais executando os mesmos métodos de serviço e repita os testes imediatamente após mudança de ACL, remoção de membro, soft-delete e restore.
- Comece a inspeção pelos clientes *unscoped*/administrativos, workers em background, migrações, constraints globais e construção de chaves de cache — são os pontos onde a identidade da requisição costuma desaparecer.

## Validation rules (aplicar antes de reportar QUALQUER achado aqui)

1. Nomeie o principal de menor confiança, o dado/estado protegido, o proprietário/tenant afetado, a cópia derivada ou operação usada e o vazamento ou mutação resultante.
2. Cite tanto a política autoritativa pretendida quanto o caminho exato no código que a omite ou contradiz, confirmando que outra camada (como RLS com `FORCE`) não neutraliza a falha.
3. Utilize exclusivamente tenants fictícios e dados sintéticos em testes locais, parando no primeiro registro cruzado observado.
4. Se políticas externas de storage S3, CDN, retenção ou backup forem decisivas e invisíveis no código, classifique como `needs_validation` descrevendo o check para o proprietário.
5. Retorne `confirmed` apenas com linhagem completa comprovada e impacto real de fronteira; retorne `needs_validation` detalhando o fato pendente de armazenamento, ACL, invalidação ou restauração.
