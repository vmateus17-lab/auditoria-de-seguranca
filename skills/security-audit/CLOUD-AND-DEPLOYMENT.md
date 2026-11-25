# Caça em Nuvem, Contêineres e Implantação (`Cloud and Deployment Hunting`)

#### When to use this file

Utilize este guia quando o repositório definir identidades de nuvem (IAM), Infraestrutura como Código (Terraform, CloudFormation, Pulumi, Helm, Ansible), contêineres Docker, manifestos Kubernetes, service mesh, funções serverless, edge workers, regras de ingress, buckets de armazenamento de objetos ou configurações específicas por ambiente. Este domínio verifica se os componentes implantados recebem exatamente a identidade, o isolamento, a alcançabilidade de rede, os segredos e as políticas pretendidas (veja também controles `INF01–INF06`). Separe rigorosamente defeitos confirmáveis no código ativo de fatos que dependem do estado vivo da conta cloud (`needs_validation`).

## Core discipline (incluir em todo prompt de agente para este domínio)

```
- Não infira exposição em produção a partir de um manifesto isolado: estabeleça qual ambiente o consome, quais overlays (Helm values, Kustomize, variáveis Terraform) o modificam e se o caminho está ativo.
- Mapeie a identidade de cada workload (IAM Role, ServiceAccount) para operações e recursos específicos. Uma política ampla só é um achado confirmado quando uma entrada de menor confiança consegue acionar uma ação não autorizada através dela.
- Ingress, proxies, service mesh, serviços de metadados (IMDS) e admission controllers são fronteiras reais, mas só contabilize um controle como garantido quando sua definição e vinculação estiverem visíveis no repositório.
- Referências a nomes de segredos não constituem vazamento de segredo: exija um leitor de menor privilégio, saída pública, artefato de build, log ou fallback inseguro.
- Use `confirmed` para configurações ativas no repositório comprovadas por renderização/avaliação local de políticas. Use `needs_validation` para políticas na conta cloud, anexos de rede, admission controllers externos ou drift em produção que exijam observação pelo proprietário.
```

## Workload identity and IAM attack classes (subagent_type: `general`)

**Workload identity overreach**
Uma identidade de pod, função serverless, edge worker ou nó possui permissões sobre tenants, contas, buckets, filas ou APIs além de seu papel, e uma entrada não confiável consegue selecionar o recurso alvo (ex.: curingas `Action: "*"` ou `Resource: "*"` combinados com seletor controlado pelo usuário).

**Cross-account or cross-tenant role confusion**
Políticas `AssumeRole`, *confused deputy* sem `ExternalId`, troca de tokens OIDC (`sub` sem filtro de repositório/branch/environment) ou políticas de recursos que aceitam afirmações de identidade não vinculadas estritamente à conta, audiência, namespace ou workload esperado.

**Application authorization delegated to cloud metadata**
Aplicação que confia em cabeçalhos de identidade, tags, labels ou IDs de conta fornecidos na requisição sem provar que vieram do plano de controle da nuvem ou de um gateway/proxy autenticado.

## Ingress, network, and control-plane attack classes (subagent_type: `general`)

**Unexpected service or management-plane reachability**
Regras de Ingress, Services `LoadBalancer`/`NodePort`, Security Groups (`0.0.0.0/0`), anotações de balanceador ou binds `0.0.0.0` que expõem rotas administrativas, portas de debug, métricas internas, bancos vetoriais (`DB07`) ou APIs de controle para redes de menor confiança.

**Trusted-proxy and mesh identity bypass**
Backend que aceita cabeçalhos `X-Forwarded-User`, sujeitos mTLS ou metadados de autorização vindos diretamente de pares fora do ingress/sidecar esperado, ou portas/rotas alternativas de healthcheck/legado que contornam o Service Mesh.

**Metadata and internal-service reachability**
Seletores de URL, webhook ou protocolo não confiáveis que alcançam metadados de instância/contêiner (`169.254.169.254`, IMDSv1 sem hop-limit/token, sockets Docker/Kubelet) carregando credenciais do workload (`APP09`).

## Container and orchestration attack classes (subagent_type: `general`)

**Host or control-plane capability exposure**
Workloads alcançáveis por entradas de menor confiança configurados com `privileged: true`, capacidades Linux perigosas (`CAP_SYS_ADMIN`, `CAP_NET_RAW`), `hostNetwork`/`hostPID`, montagens `hostPath`, socket do container runtime (`/var/run/docker.sock`) ou tokens `automountServiceAccountToken` com RBAC elevado no cluster (`INF01`).

**Admission and policy path inconsistency**
Um fluxo principal de deploy aplica políticas restritas de imagem, namespace e privilégio, enquanto um job secundário, cronjob, migração, restauração ou controller alternativo contorna essas restrições.

**Namespace and label trust confusion**
`NetworkPolicy`, políticas de admissão (Kyverno/OPA), injeção de segredos ou federação de identidade baseadas em labels, anotações ou nomes de recursos que um ator de menor privilégio consegue definir ou alterar.

## Configuration and secret lifecycle attack classes (subagent_type: `general`)

**Security-control precedence drift**
Valores default de charts Helm, overlays por região, variáveis de ambiente ou flags de linha de comando que desativam autenticação, TLS, isolamento de tenant ou auditoria em um ambiente implantado ao sobrescrever o arquivo base. Renderize sempre a configuração final combinada de cada ambiente mantido.

**Secret exposure across workload boundaries**
Segredos reais persistidos em variáveis de build público, logs de inicialização, argumentos de linha de comando visíveis em `/proc`, volumes compartilhados amplos, imagens publicadas ou endpoints de diagnóstico legíveis por outros workloads/tenants (`INF04`).

**Credential renewal and outage fallback**
Falhas ao montar, renovar ou rotacionar credenciais efêmeras (Vault/IRSA) que levam a aplicação a manter permissões revogadas ativas indefinidamente ou cair em modo *fail-open* sem autenticação.

## Managed storage, events, and edge attack classes (subagent_type: `general`)

**Object and signed-URL policy confusion**
Políticas de buckets S3/GCS/R2, chaves de objetos, origens de CDN ou *presigned URLs* que deixam de vincular principal, método HTTP (`GET` vs `PUT`), prefixo de tenant, audiência ou expiração curta.

**Event-source identity confusion**
Funções serverless ou workers que confiam em campos internos do JSON de um evento (SNS, EventBridge, Pub/Sub, webhooks cloud) como identidade de origem sem validar o envelope assinado pelo provedor, tópico/assinatura, conta, região e proteção contra replay.

**Edge/runtime boundary mismatch**
Workers de borda (*edge runtimes*) e servidores de origem assumindo políticas divergentes de autenticação, sanitização de headers, isolamento de tenant ou cache, onde o fallback para a origem altera a autoridade efetiva.

## Universal moves (aplicar transversalmente neste domínio)

- Renderize localmente todos os ambientes mantidos (`helm template`, `kubectl kustomize`, `terraform plan` offline) e construa uma matriz cruzando porta externa, identidade do workload, pares de rede, segredos montados e recursos cloud acessados.
- Siga uma requisição, objeto, label ou evento de menor confiança até a política cloud final, identificando qual credencial de workload executa a operação e qual condição deveria restringi-la.
- Compare caminhos normais de deploy com jobs de migração, backup/restore, manutenção e failover.

## Validation rules (aplicar antes de reportar QUALQUER achado aqui)

1. Estabeleça o caminho ativo no código e o objeto final renderizado de implantação; caso contrário, use `needs_validation` indicando qual manifesto, variável ou anexo na conta cloud falta observar.
2. Nomeie o chamador/workload de menor confiança, a identidade cloud/aplicação usada, o seletor controlável, o recurso afetado e a operação ou exposição não autorizada.
3. Verifique os padrões nativos do provedor/orquestrador na versão fixada, sem presumir IPs públicos ou firewalls abertos ausentes do código.
4. Valide localmente apenas por renderização de templates, avaliadores estáticos de políticas (OPA/Checkov/Trivy) ou emuladores isolados com identidades fictícias — jamais sonde contas cloud reais.
5. Retorne `confirmed` somente com trace ativo completo e resultado concreto de fronteira; retorne `needs_validation` detalhando a política IAM, anexo de rede ou drift que o proprietário deve verificar no painel/CLI da nuvem.
