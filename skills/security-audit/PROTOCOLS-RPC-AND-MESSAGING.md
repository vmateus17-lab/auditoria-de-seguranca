# Caça em Protocolos, RPC e Mensageria (`Protocols, RPC, and Messaging Hunting`)

#### When to use this file

Utilize este guia quando o alvo empregar gRPC, transportes GraphQL, Cap'n Proto, Thrift, Protobuf, protocolos binários customizados, streaming RPC, webhooks, message brokers (RabbitMQ, Kafka, NATS, SQS/SNS), filas, pub/sub ou barramentos de eventos. Este domínio audita identidade de pares, interpretação lógica de mensagens, roteamento, replay, ordenação e semântica de entrega (veja também controles `APP16–APP17` e `PRO03`). Combine com `MEMORY-SAFETY-AND-BINARY.md` para segurança de memória em parsers nativos, `WEB-PROTOCOL-AND-AUTH.md` para framing HTTP e `RESOURCE-EXHAUSTION-AND-AVAILABILITY.md` para impacto de disponibilidade.

## Core discipline (incluir em todo prompt de agente para este domínio)

```
- "Rede interna" não é autenticação. Nomeie a identidade do par (peer identity) em cada salto e mostre como ela se transforma no principal de aplicação usado na autorização.
- Validação de schema comprova formato sintático da mensagem, não proveniência, autoridade sobre o recurso, ordenação nem valores seguros. Siga os campos decodificados até a política e os efeitos colaterais.
- Garantias do broker (at-least-once, at-most-once) diferem das garantias da aplicação. Documente retentativas, ordenação, ACK, deduplicação e escopo transacional antes de julgar mutações de estado.
- Divergências de parser exigem dois consumidores concretos, versões de schema ou representações na rede e um valor divergente relevante para segurança.
- Use `confirmed` para caminhos completos no código somados a testes locais delimitados produtor/consumidor. Use `needs_validation` para ACLs externas do broker, identidade de Service Mesh ou vínculos de tópicos invisíveis no repositório.
```

## Framing, schema, and interpretation attack classes (subagent_type: `general`)

**Message boundary and canonicalization disagreement**
Componentes divergem quanto a tamanho de mensagem, compressão, campos duplicados, campos desconhecidos (*unknown fields* em Protobuf), codificação, largura numérica (int32 vs int64/float JS) ou precedência entre envelope e corpo. Compare gateways HTTP-para-gRPC, bindings de diferentes linguagens e conversores de versão.

**Union, enum, and default confusion**
Variantes `oneof` desconhecidas, discriminadores ausentes, valores zero (`0` default em Protobuf v3), privilégios default ou mapeamentos de retrocompatibilidade alcançam código que pressupunha um caso estritamente validado.

**Envelope and payload identity mismatch**
O middleware de autorização valida metadados confiáveis do cabeçalho/envelope, mas o handler executa a operação usando um `tenant_id`, `user_id` ou remetente conflitante extraído do corpo da mensagem.

## RPC identity and authorization attack classes (subagent_type: `general`)

**Interceptor and method-path inconsistency**
Um interceptor de autenticação/autorização protege chamadas unárias, mas falha em métodos de streaming, Server Reflection, healthchecks estendidos, rotas transcodificadas por gRPC-Gateway ou mensagens individuais subsequentes dentro de um stream aberto.

**Peer identity to application-principal confusion**
mTLS, workload identity ou credenciais do broker autenticam o canal entre serviços, mas um campo controlado pelo cliente seleciona arbitrariamente o usuário final ou tenant sem política determinística de vinculação (*confused deputy* entre microsserviços).

**Per-item and streaming authorization gaps**
Um stream bidirecional, assinatura pub/sub ou lote (*batch*) é autorizado apenas na abertura da conexão, permitindo que itens subsequentes referenciem recursos de outros tenants ou continuem operando após revogação de token/papel.

**Callback and reply-correlation confusion**
`correlation_id` ou `reply_to` previsíveis, reutilizados ou sem escopo de tenant permitem que uma resposta, webhook ou cancelamento satisfaça a requisição pendente de outro chamador.

## Broker and queue isolation attack classes (subagent_type: `general`)

**Topic, routing-key, and subscription scope gaps**
Um produtor ou consumidor consegue publicar ou assinar tópicos de outro tenant, curingas (`*`, `#` em MQTT/AMQP), grupos de consumidores, partições ou filas de resposta por interpolação insegura de nomes de tópicos.

**Dead-letter, retry, and diagnostic disclosure**
Mensagens encaminhadas para *Dead-Letter Queues* (DLQ), tópicos de erro, traces OpenTelemetry ou painéis operacionais retêm segredos, tokens ou PII cross-tenant acessíveis a consumidores de menor privilégio.

**Untrusted producer treated as control plane**
O corpo de uma mensagem na fila consegue autodeclarar-se evento administrativo, callback de pagamento ou comando de migração sem assinatura criptográfica independente ou validação estrita da identidade do produtor.

## Replay, ordering, and transaction attack classes (subagent_type: `general`)

**Duplicate delivery and idempotency gaps**
Entregas `at-least-once` ou retentativas automáticas repetem transferências, créditos ou disparos externos porque falta chave atômica de idempotência escopada por tenant e operação (`APP14`).

**Out-of-order and stale message acceptance**
Eventos atrasados, permissões pré-revogação ou estados antigos chegam após eventos mais novos e sobrescrevem o estado atual por ausência de checagem de versão/sequência monotônica (*optimistic concurrency* / *tombstones*).

**Acknowledgment/commit ordering defects**
Enviar `ACK` ao broker antes do commit durável no banco (perdendo eventos críticos de auditoria/revogação) ou commitar mutação não idempotente antes de um `ACK` sujeito a falha sem padrão *Transactional Outbox/Inbox*.

**Partial multi-consumer transitions**
Múltiplos consumidores implementam juntos uma transição crítica de segurança/negócio, mas falhas parciais deixam o sistema em estado inconsistente ou semi-autorizado.

## Universal moves (aplicar transversalmente neste domínio)

- Desenhe o fluxo `produtor → broker/transporte → gateway → consumidor → banco` para cada família de mensagens, anotando em cada salto o par autenticado, o campo autoritativo de tenant/recurso, a validação e o efeito colateral.
- Envie a mesma fixture mínima para cada versão de schema ou binding presente no repositório, testando campos duplicados, ausentes, zero, fora de ordem e reenviados (*replay*) em memória, sem gerar carga.
- Compare rotas normais com retentativas, DLQs, reflection, streams longos e gateways transcodificados.

## Validation rules (aplicar antes de reportar QUALQUER achado aqui)

1. Nomeie o produtor/par realista, a mensagem aceita, a identidade autenticada do canal, o principal/recurso afetado e a mutação ou vazamento não autorizado.
2. Em divergências de interpretação, cite ambos os parsers/consumidores e o valor decodificado divergente.
3. Em falhas de replay/ordenação, estabeleça o modelo real de entrega e reproduza a quebra do invariante em transporte local/em memória.
4. Verifique todos os interceptors, ACLs de broker visíveis no código, gateways e consumidores; anexos externos invisíveis exigem `needs_validation`.
5. Retorne `confirmed` apenas com ciclo completo da mensagem e impacto concreto demonstrado; retorne `needs_validation` especificando a ACL de broker, identidade de mesh ou topologia necessária.
