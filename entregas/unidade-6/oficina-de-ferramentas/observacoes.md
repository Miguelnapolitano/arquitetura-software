# Observações — Oficina de ferramentas: Docker, kind e Kubernetes locais

## Condição alterada

A oficina define a condição a tornar observável: o Deployment `hospital-api`
deve partir de **nenhum recurso no namespace `hospital`**, e o cluster kind deve
ser **criado do zero e descartável**. Alterei apenas a "configuração" do
experimento — a tag de imagem `hospital-api:imagem-propositalmente-ausente`, que
**não é imagem real** e **não toca nada compartilhado** — para observar a
atualização bloqueada e o rollback. Não editei manifest nem arquivos do
repositório, não usei `sudo` para apontar kubectl a outro contexto e não toquei
em recursos de outros estudos.

## Instalação de ferramentas

O ambiente era Linux com Docker 29.8.1 em execução, mas **kind e kubectl não
estavam instalados**. Instalei os binários localmente (sem `sudo`, sem tocar o
socket Docker), em `~/.local/bin`:

- `kind` v0.27.0 (`kind-linux-amd64`);
- `kubectl` v1.31.4 (`kubernetes-client-linux-amd64`).

Ambos fora do caminho padrão, então registrei `export PATH="$HOME/.local/bin:$PATH"`
no shell usado na oficina. `docker version` com `python3 --version` já atendiam;
o `.venv` tem Python 3.12.12 com `pytest` e `pyyaml` (ver `evidencias/versoes.txt`).

## O que a saída revelou

### Reconciliação declarativa em duas réplicas prontas

`kubectl apply` (namespace primeiro, depois os quatro recursos que pertencem a
`hospital`) retornou `created` para todos; `rollout status` convergiu para
"successfully rolled out", a lista `-o wide` mostrou `deployment.apps/hospital-api
2/2 … AVAILABLE 2`, dois Pods `Running 1/1` nos IPs `10.244.0.5/.6`, e o
EndpointSlice do Service passou a conter exatamente esses dois endereços. A
sequência provou a **ordem de criação** (namespace antes dos recursos) e a
**cobertura dos endpoints** por readiness — o `curl` em `127.0.0.1:18080` só
funcionou porque um Pod pronto estava no EndpointSlice.

O que a saída **não** prova: atendimento real em produção, tolerância a falha de
zona ou autoscaling efetivo — todos fora do escopo do laboratório.

### O dry-run do kubectl 1.31 depende do servidor

O primeiro `kubectl apply --dry-run=client -f infra/k8s/namespace.yaml` falhou com
`failed to download openapi … connection refused`: o cliente 1.31 busca o schema
OpenAPI **junto ao servidor** para validar, e ainda não havia cluster. Em vez de
"consertar" a saída, registrei o comportamento real (`evidencias/dryrun.txt`),
usei a contingência estática da oficina (`--validate=false` para checar sintaxe no
cliente + `pytest`) e, **após criar o cluster**, rodei o dry-run original:
`namespace/hospital created (dry run)` etc. (`evidencias/dryrun-cluster.txt`). A
lição é a da própria oficina: dry-run confirma sintaxe aceita, não executa Pods.

### Imagem ausente ≠ falha de readiness

A troca de tag para `imagem-propositalmente-ausente` gerou `image updated`, o
rollout esgotou o timeout de 20 s e o Pod novo entrou em `ImagePullBackOff`
(`evidencias/rollback-falha.txt`). O `describe` do Pod fechou o ciclo:
`ErrImagePull → Failed to pull image … pull access denied … → ImagePullBackOff`
(`evidencias/describe-pod-imagepullbackoff.txt`). Em contraste, `describe` do
Deployment mostrou `OldReplicaSets: hospital-api-57568c78f9 (2/2 replicas created)`
e `NewReplicaSet: hospital-api-55b5b865cf (1/1 replicas created)` —
`maxUnavailable: 0` manteve as réplicas antigas de pé. Isso separa as duas falhas
que a oficina pede comparar: **falha de imagem** (nem inicia; evidência é
`ErrImagePull`) vs **falha de readiness** (inicia, mas não entra no Service;
evidência é o Pod ficar fora do EndpointSlice). Ambas bloqueiam uma atualização,
mas a correção é distinta: imagem → nova tag válida; readiness → corrigir o
check, não reiniciar o Pod.

### Rollback e liveness

`rollout undo` devolveu `dep.apps/hospital-api rolled back` e
"successfully rolled out"; a imagem voltou a `hospital-api:1.0.0`
(`evidencias/imagem-pos-undo.txt`), os dois Pods antigos voltaram a `Running`, e
`curl /health/live` respondeu `{"status":"live"}`. O rollout history registrou
revisões 2 e 3 (a revisão 1 é a base do kind com o cluster recém-criado). O undo
não desfez a revisão indevida, apenas restaurou a saudável — e provou que o
rollback é **contenção de runtime**, não reversão de banco.

### HPA: político declarado, métrica ausente

Analisando a resposta da pergunta "o que o laboratório não prova": o HPA exibiu
`cpu: <unknown>/70%` e `describe` mostrou `AbleToScale True` mas
`ScalingActive False — FailedGetResourceMetric: the server could not find the
requested resource (get pods.metrics.k8s.io)` (`evidencias/hpa-describe.txt`). Ou
seja: o manifesto declara a **política** (2–5 réplicas, alvo 70% de CPU sobre o
`request`), mas sem Metrics Server no kind não há **prova** de aumento automático
sob carga. Registrei o fato em vez de inventar escalonamento — esta é uma das duas
conclusões-limite que a evidência sustenta.

## Questões exploratórias respondidas

- **Que risco existe em executar `kubectl apply` no contexto errado?** Aplicar
  recursos em um contexto compartilhado/rede da turma criaria o namespace e o
  Deployment onde outra pessoa trabalha — conflito, sobreescrita e exposição. Por
  isso a oficina exige conferir `current-context` e nunca usar `sudo` para mudar
  o alvo de kubectl.
- **Por que o laboratório fixa o acesso em `127.0.0.1`?** O `extraPortMappings`
  do kind limita o hostPort a `listenAddress: 127.0.0.1` — o NodePort 30080 só é
  alcançável pelo console local; nenhum acesso externo é criado.
- **Por que `IfNotPresent` faz sentido para a imagem carregada no kind?** A
  imagem existe apenas no nó kind após `kind load`; `IfNotPresent` evita pull
  desnecessário e depende da revisão local. Para produção, uma tag exclusiva/digest
  no registry tornaria a política mais segura — cores da mesma lição.
- **Que evidência adicional uma CI produziria para uma imagem de produção?**
  Prova de origem e conteúdo: assinatura/attestations, scan de vulnerabilidades,
  SBOM, hash, teste de imagem (`docker run` + health) e armazenamento público/
  privado com permissões — não chegaria por "agora passa no kind".
- **Qual falha serait detectada por readiness mas não deveria reiniciar um
  processo?** Uma falha de dependência temporária (ex.: banco/API externa em
  manutenção) que o `/health/ready` vê mas o processo aguenta; tirar o Pod do
  Service é a medida certa, reiniciá-lo a cada `failureThreshold` agitaria o
  rollout sem corrigir a causa.
- **Por que duas réplicas no kind não equivalem a duas zonas?** O cluster tem um
  único nó (control-plane em contêiner no host Docker); as duas réplicas dividem
  CPU/disco/rede do mesmo nó físico. Disponibilidade de infra não é provada; a
  reconciliação e a atualização gradual, sim.
- **Que parte de uma migração de banco `rollout undo` não desfaria?** Qualquer
  efeito externo persistido por uma revisão nova — schema, dados, filas — não
  é revertido só por trocar a imagem de volta. Rollback restaura o Pods, não o
  estado do banco.
- **Que política de CI evitaria chegar a uma tag inexistente?** Publicar imagem
  com digest/tag única confirmada (build+push como passo gate), proibir `latest`
  e validar que a tag referenciada existe no registry antes do `kubectl set
  image`; o experimento só acontece porque a política permitiu inventar uma tag.
- **Como requests de CPU participam do cálculo de utilização do HPA?** A
  utilização é computada sobre o `requests.cpu` (aqui `100m`), com limite de
  `250m`; o `70%` apontado é a razão sobre o request. É por isso que requests
  importam: definem a referência da métrica, não só a reserva.
- **Que carga sintética respeitaria a capacidade da máquina do grupo?** Uma carga
  de CPU contida (requisições concorrentes moderadas) medida antes/depois; no kind
  sem Metrics Server ela não moveria o HPA, pois a métrica nem chega ao
  `metrics.k8s.io` — a limitação ficou registrada.
- **Que métrica além de CPU indicaria uma fila crescente?** Métrica de backlog da
  fila de elegibilidades ou de mensageria — um HPA baseado em `kafka_consumer_lag`
  / fila (via Prometheus adapater ou KEDA) responderia antes da CPU, que é o
  exemplo educacional do manifesto.

## Divergências e notas

- **Dry-run antes do cluster:** com kubectl 1.31, `--dry-run=client` precisa do
  schema do servidor; sem cluster ele falha. Documentado em vez de escondido
  (`dryrun.txt` vs `dryrun-cluster.txt`), e a validação estática foi completada
  por `--validate=false` + `pytest`.
- **HPA `<unknown>`:** sem Metrics Server, o alvo de CPU ficou desconhecido. Não
  instalei add-ons durante a aula (escopo do laboratório); registrei o fato
  (`hpa-describe.txt`) — a oficina pede exatamente esse registro.
- **Imagem local mantida:** apaguei o cluster com `kind delete cluster --name
  hospital-local` mas **mantive** `hospital-api:1.0.0` no Docker para a próxima
  aula (a oficina permite). `docker ps` após a limpeza não lista contêineres do
  kind (os contêineres `minio`/`konga` visíveis não pertencem a esta oficina e
  não foram tocados).
- **PATH:** kind/kubectl em `~/.local/bin`; a sessão usou
  `export PATH="$HOME/.local/bin:$PATH"`.

## Responsabilidade arquitetural sustentada

A oficina liga o ciclo **build → cluster → manifests → sondas → rollback** à
prática declarativa de nuvem:

1. **Imagem imutável como contrato**: `Dockerfile` (Python 3.12, usuário sem
   privilégios, porta 8000) e `docker image inspect` (User=app, linux/amd64)
   tornam a origem verificável; a tag local não é registry de produção.
2. **Cluster descartável como restrição de escopo**: `cluster.yaml` fixa um único
   nó, porta `127.0.0.1:18080`/NodePort 30080; nenhum comando alcança contexto
   compartilhado e a limpeza remove tudo.
3. **Estado declarado reconciliado**: manifests convergem (2 réplicas, resources,
   `RollingUpdate`), probes separam pronto-de-receber (`/health/ready`) de
   vivo (`/health/live`), Service seleciona por `app: hospital-api` e o HPA
   declara política 2–5.
4. **Contenção observada**: tag ausente → `ImagePullBackOff` → rollout bloqueado
   sem derrubar os Pods antigos (`maxUnavailable: 0`) → `rollout undo` restaura a
   revisão saudável — evidência de que o Kubernetes não conserta imagens inválidas
   sozinho, e rollback é procedimento de runtime, não de dados.
5. **Limite honesto**: tolerância a zona, autoscaling sob carga e prontidão de
   produção são explicitamente o que o laboratório **não** prova.