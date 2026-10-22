### HLD: n3x-sidecar (entrypoint, bootstrap e core do `.so`)

Versão: 1.1
Data: 2026-10-20
Responsável: Guilherme Giusti

---

### Objetivo técnico
Descrever como o sidecar do N3X sobe a partir do `n3x.yaml`, recebe tráfego HTTP/1.1, HTTP/2 e gRPC e executa os plugins dentro do Envoy num único `libn3x.so`, contendo as falhas de cada plugin e mantendo a conexão com o controller sempre fora do caminho da requisição.

Este HLD é dono destes contratos (conforme `docs/hld/HLDs-N3X.md`):
- Schema do `n3x.yaml` do sidecar, incluindo a seção `plugins`
- Bootstrap do Envoy gerado pelo entrypoint e `EnvoyFilter` estático do modo Istio
- Portas do sidecar (já fixadas na faixa 17xxx pelo ADR-006 do `n3x-sdk`)
- Comportamento do executor da cadeia, da contenção de falhas e da conexão compartilhada

Problemas técnicos endereçados
- O `.so` precisa rodar em dois alvos com APIs e barreiras de panic diferentes: parcial no Envoy 1.40 e inexistente no istio-proxy 1.31
- A iteração de filtros do Envoy é reimplementada dentro de um único filtro, com seguir, parar, retomar e grupos paralelos
- Uma negação gRPC não pode chegar ao cliente como `grpc-status 0`
- O runtime Go divide CPU e memória com o Envoy no mesmo cgroup
- Plugin que ignora o `ctx` mantém goroutines vivas depois do prazo, porque Go não mata goroutines por fora
- Nenhum segredo decifrado pode parar no bootstrap, no `/config_dump` ou em log

Base: `docs/prd/PRD-N3X.md` versão 1.5, de 2026-10-20. Requisitos cobertos: FR-001, FR-002 (lado do sidecar), FR-003 e FR-004, mais os requisitos não funcionais de desempenho, confiabilidade, segurança e observabilidade do sidecar. Metas do sistema divididas por componente conforme o ADR-010 do `n3x-sdk`.

Dependências com outros sistemas
- `n3x-sdk`: contratos de plugin de proxy, oito fases, políticas de falha, padrão de IDs e manifesto do plugin (só referenciados aqui)
- `n3x-plugin-core`: login, sessão, interceptor do token, registro, heartbeat, eventos e formato da config efetiva com mascaramento e hash (só referenciados aqui; ADR-005, ADR-006 e ADR-007 do `n3x-plugin-core`)
- `n3x-builder`: importa o módulo `core/` deste repositório, gera o `libn3x.so` por alvo e arquitetura, embute o manifesto e aplica a análise estática
- `n3x-controller`: destino da conexão gRPC compartilhada (`17443`, TLS 1.3)
- Imagens base `envoyproxy/envoy:v1.40.x` e `istio/proxyv2:1.31.x`, fixadas por digest
- SDK Go de dynamic modules do Envoy, em pseudo-versão fixada pelo catálogo de alvos do builder
- Orquestradores: Kubernetes 1.29 ou superior (sidecar nativo, ADR-007 do `n3x-sdk`), ECS e docker compose
- Istio 1.31, apenas como compatibilidade (ADR-006 do `n3x-sdk`)

**Clarificações (2026-10-22)**
Alinhamentos aprovados na validação de consistência com o PRD e as FDDs SID-01 a SID-12 (o texto das seções abaixo já reflete cada um):
- Admin do Envoy em socket Unix `/tmp/n3x/admin.sock`, sem porta TCP; a `17001` fica reservada e sem uso (SID-06)
- Config do filtro só com `config_path` e com `do_not_close: true` (SID-01 e SID-06; nota de revisão no ADR-007)
- Imagem com `ENTRYPOINT ["/usr/local/bin/n3x"]` e `CMD ["run"]`; subcomandos `probe`, `help`, `istio-drain` e a palavra `proxy` do modo Istio (SID-01, SID-08, SID-09 e SID-11; nota de revisão no ADR-003)
- Drenagem: os 2 s dos plugins começam antes do SIGTERM ao Envoy, pelo canal de controle; total máximo de `runtime.drain_timeout` mais 9 s (SID-09; nota de revisão no ADR-006)
- Prazo de requisição gRPC pelo `grpc-timeout` do cliente, até 1 h; teto das chamadas ao controller configurável em `controller.timeouts.call` (SID-06 e SID-10; nota de revisão no ADR-006)
- Até 5 entradas do Envoy no Go com trabalho da cadeia por requisição (SID-02; nota de revisão no ADR-009)
- `OnComplete` para toda instância do recorte; saída do entrypoint com o código do Envoy quando ele morre em serviço (SID-03 e SID-07)
- Orçamento ocioso em MiB e com a parcela do entrypoint (SID-07, SID-09 e SID-12; nota de revisão no ADR-009)
- Campos `runtime.request_body` e `runtime.plugin_limits` no schema e métricas novas, com o catálogo completo na SID-02 (SID-01, SID-02 e SID-04)
- Decisões pendentes resolvidas pelas FDDs, marcadas na lista do fim deste documento

---

### Arquitetura geral
O sidecar é um container com dois processos e duas entregas de código.

**Entregas do repositório** (ADR-001)
- **Binário `n3x`** (`cmd/n3x`): entrypoint embutido na imagem `ghcr.io/giustech/n3x/sidecar:<semver>-<alvo>`
- **Módulo Go `core/`**: executor da cadeia, contenção de falhas, conexão compartilhada, telemetria e adaptadores por alvo. Não é compilado aqui como `.so`: o `n3x-plugin build` do builder gera um `main` que importa `core/`, o plugin core e os plugins escolhidos e compila um único `libn3x.so`. Os pacotes `core/config` e `core/identity`, e o pacote `secret` do `n3x-sdk` (ADR-014 do `n3x-sdk`, que substituiu `core/secret`), são usados tanto pelo entrypoint quanto pelo `.so`, com a mesma validação dos dois lados

```
n3x-sidecar/
├── cmd/n3x/                  entrypoint: run, validate, encrypt, keygen
├── internal/supervisor/      ciclo de vida do Envoy, sinais e drenagem
├── internal/bootstrap/       geração do bootstrap (golden em testdata/)
├── internal/health/          servidor 17021 e health check da aplicação
├── core/config/              schema do n3x.yaml v1 e validação
├── core/identity/            nome do pod ou ID do container
│                             ENC[...] vem do pacote secret do n3x-sdk (ADR-014 do n3x-sdk)
├── core/chain/               cadeia compilada, fases, grupos e recortes
├── core/fault/               recover, políticas, disjuntor e zumbis
├── core/grpcconn/            conexão compartilhada com o controller
├── core/telemetry/           stats do Envoy e logs JSON
├── core/adapter/envoy140/    build tag n3x_envoy140
├── core/adapter/istio131/    build tag n3x_istio131
├── deploy/istio/             EnvoyFilter estático
├── images/                   Dockerfiles das imagens de runtime
└── bench/                    benchmark open-loop e Envoy puro de referência
```

**Topologia em runtime (modo padrão: Kubernetes, ECS e docker compose)**
```
                 container do sidecar (UID 17000)
  cliente ──► :17000 ┌──────────────────────────────────────────┐
                     │ n3x run (PID 1, supervisor)              │
  orquestrador ─► :17021  health /live e /ready                  │
                     │   └─ filho: envoy --concurrency N        │
                     │        listener 17000 → filtro n3x       │
                     │        └─ libn3x.so (runtime Go único)   │ ──► controller :17443
                     │             core + plugin core + plugins │     (gRPC, TLS 1.3)
  Prometheus ──► :17090  listener → admin /stats/prometheus    │
                     │        admin /tmp/n3x/admin.sock         │
                     └──────────────────┬───────────────────────┘
                                        ▼ 127.0.0.1:<porta>
                                    aplicação
```

**Sequência de subida (`n3x run`)** (FR-001), com meta de pronto em até 5 s
1. Lê `/etc/n3x/n3x.yaml` ou `--config` e valida o schema v1 (`core/config`)
2. Confere que o alvo do manifesto embutido no `.so` é o mesmo da imagem (`/etc/n3x/target`), confere a integridade do `.so` (SHA-256 das seções ELF carregáveis, refeito com `debug/elf`; ADR-002 do `n3x-builder`), recusa `format` de manifesto maior do que conhece e confere a versão do `core/` gravada no manifesto: aceita a mesma major com minor igual ou menor que a da imagem; fora disso, encerra dizendo qual versão de imagem usar (ADR-001 do `n3x-builder`)
3. Valida cada item de `plugins` contra o schema do plugin no manifesto, lido da seção ELF do `.so` sem carregá-lo
4. Lê a chave AES do arquivo montado e decifra os `ENC[...]` para validar; nada decifrado sai do processo
5. Detecta a identidade da instância (`core/identity`)
6. Calcula `--concurrency`, `GOMAXPROCS`, `GOMEMLIMIT` e `GODEBUG=cgocheck=0` a partir do cgroup e de `resources` (ADR-008)
7. Gera o bootstrap em `/tmp/n3x/bootstrap.yaml`, sem nenhum segredo (ADR-007)
8. Roda `envoy --mode validate` (prazo de 10 s)
9. Inicia o Envoy como processo filho, com as variáveis do runtime Go, e passa a repassar sinais
10. Abre o `17021`; `/ready` só responde 200 quando o Envoy está pronto e a aplicação passa no health check (prazo de 30 s para o Envoy ficar pronto, senão encerra)

Qualquer falha nos passos 1 a 8 encerra o container com código diferente de zero e mensagem que aponta o campo, sem segredos. Se o Envoy morre fora da drenagem, o entrypoint sai com o código do Envoy (128 mais o número do sinal, se morto por sinal; 0 vira 1, para o orquestrador tratar como falha; SID-07); quem reinicia é o orquestrador.

**Drenagem no SIGTERM** (FR-003; detalhada na SID-09)
1. `/ready` passa a responder 503 na hora
2. O entrypoint avisa o `.so` pelo canal de controle local (socket Unix `/tmp/n3x/control.sock`, modo 0600, servido pelo `.so`), e o plugin core envia o evento de início da drenagem
3. `POST /drain_listeners?graceful&inboundonly` no admin, com o Envoy iniciado com `--drain-strategy immediate`: o Envoy pede o fechamento das conexões (`GOAWAY` em HTTP/2 e `Connection: close` em HTTP/1.1), e o listener de métricas `17090` continua servindo
4. Espera as conexões ativas chegarem a zero, depois de uma espera mínima de 3 s, ou `runtime.drain_timeout` (padrão de 20 s)
5. Pede ao `.so`, pelo canal de controle, o fim dos plugins: as tarefas de fundo têm 2 s para encerrar e o plugin core avisa a saída ao controller com o Envoy ainda vivo; o SIGTERM ao Envoy só sai depois da confirmação ou em 3 s
6. Manda SIGTERM ao Envoy; sem saída em 5 s, SIGKILL

Prazo total máximo do sinal à saída: `runtime.drain_timeout` mais 9 s (29 s com o padrão). A aplicação pode antecipar os passos 1 a 4 com o gatilho opcional `POST http://127.0.0.1:17021/drain`, aceito só de loopback (SID-09).

**Modo Istio** (ADR-006 do `n3x-sdk`: o Istio manda)
- A imagem `ghcr.io/giustech/n3x/sidecar:<semver>-istio-1.31` é baseada no `proxyv2` e substitui a imagem injetada (anotação `sidecar.istio.io/proxyImage`)
- Quando o binário `n3x` recebe os argumentos do injetor (`proxy sidecar ...`), ele entra no modo Istio: lê e valida o mesmo `n3x.yaml`, calcula `GOMAXPROCS`, `GOMEMLIMIT` e `GODEBUG` e faz `exec` do `pilot-agent` com os mesmos argumentos (detecção pelo primeiro argumento `proxy`, com os demais subcomandos do `pilot-agent` repassados direto; SID-11)
- Bootstrap, portas, health (`15021`) e drenagem são do Istio; o N3X não abre `17000`, `17021` nem `17090`. As seções `listener` e `app` do `n3x.yaml` são ignoradas, com aviso no log
- O `EnvoyFilter` estático insere o filtro do N3X antes do router nos listeners de entrada, apontando para o `n3x.yaml` montado no pod
- `GOMAXPROCS` precisa ser maior ou igual ao `concurrency` do proxy definido no Istio; o entrypoint lê o valor, recusa subir se o `GOMAXPROCS` calculado ficar abaixo dele e registra os valores efetivos no log
- As métricas `n3x_` saem pelo merge de métricas do Istio, desde que o prefixo esteja incluído (hipótese: `proxyStatsMatcher` com `inclusionPrefixes: ["n3x_"]`)
- A conexão do `.so` com o controller sai direto, porque o Istio não intercepta o tráfego do UID `1337` do proxy

Ambiente de implantação
- Agnóstico: cloud ou on-premises, onde houver containers (topologias de referência em `n3x-sdk/deploy/`, ADR-008 do `n3x-sdk`)
- Kubernetes 1.29 ou superior: sidecar nativo (`initContainers` com `restartPolicy: Always`), probes em `17021/live` e `17021/ready`, `n3x.yaml` por ConfigMap e chave AES por Secret montado como arquivo
- ECS: sidecar e aplicação na mesma task (`awsvpc`), aplicação com `dependsOn` do sidecar, health check do container por comando (`n3x probe live`, SID-09)
- docker compose: aplicação com `network_mode: "service:<sidecar>"`, escutando em `127.0.0.1`
- Istio: imagem `istio-1.31` no lugar do `proxyv2` injetado, `n3x.yaml` e chave montados no container do proxy (hipótese: anotação `sidecar.istio.io/userVolumeMount`)

Tecnologias principais
- `envoyproxy/envoy:v1.40.x` e `istio/proxyv2:1.31.x`, fixadas por digest
- Dynamic modules do Envoy com o filtro HTTP estável e o SDK Go oficial, em pseudo-versão fixada por alvo
- Go 1.26 ou superior; adaptadores separados por build tag (`n3x_envoy140` e `n3x_istio131`)
- grpc-go com TLS 1.3 para a conexão compartilhada
- `log/slog` com saída JSON no entrypoint e no core
- Stats nativas do Envoy (counter, gauge e histogram) para as métricas `n3x_`
- wrk2, Nighthawk ou ghz com HdrHistogram no benchmark

Padrões adotados
- Supervisor de processo: entrypoint como PID 1, Envoy como filho
- Core e adaptador por alvo: o executor não importa o SDK do Envoy; só `core/adapter/*` importa
- Cadeia de filtros em oito fases fixas dentro de um único filtro, com grupos paralelos por `order` (ADR-005)
- Config estática e imutável, lida pelo próprio `.so` (ADR-007)
- Prazo obrigatório em toda operação (ADR-006)
- Disjuntor por instância de plugin (ADR-004)
- Backpressure por limite de trabalhos simultâneos e buffer limitado
- Degradação graciosa: o controller nunca está no caminho da requisição

---

### Componentes e responsabilidades
| Componente | Responsabilidades | Dependências |
| ----------- | ----------------- | ------------ |
| Entrypoint `n3x` (`cmd/n3x`) | Subcomandos `run`, `validate`, `encrypt`, `keygen`, `probe`, `help` e, na imagem `istio-*`, `istio-drain` (ADR-003 e nota de revisão); detecção do modo Istio pela palavra `proxy`; exit codes e mensagens sem segredos | `core/config`, `secret` do SDK, `core/identity` |
| Supervisor (`internal/supervisor`) | Cálculo do runtime Go e do `--concurrency`; inicia e supervisiona o Envoy; repassa sinais; conduz a drenagem; avisa o `.so` pelo canal de controle local | Envoy, admin em `/tmp/n3x/admin.sock` |
| Gerador de bootstrap (`internal/bootstrap`) | Gera o bootstrap sem segredos (listener, cluster, normalização, limites, overload manager, filtro único, listener de métricas e admin local); valida com `envoy --mode validate` | `core/config` |
| Health (`internal/health`) | Serve `17021` (`/live` e `/ready`); executa o health check da aplicação (`http`, `grpc` ou `tcp`); vira "não pronto" na drenagem (ADR-002) | Admin do Envoy (`/ready`), aplicação |
| Config (`core/config`) | Schema v1 do `n3x.yaml`; validação contra o schema de cada plugin do manifesto; regra de grupos paralelos; mesma validação no entrypoint e no `.so` | Manifesto embutido no `.so` |
| Segredos (`secret` do `n3x-sdk`) | Decifra `ENC[...]` com AES-256-GCM e a chave em arquivo; `encrypt` e `keygen`; zera buffers após o uso (melhor esforço) | Chave AES montada |
| Identidade (`core/identity`) | Nome do pod com hash no Kubernetes; ID do container no ECS (metadata v4) e no docker compose (cgroup ou `mountinfo`); hostname como último recurso, com aviso | Ambiente do container |
| Envoy | Listeners, normalização de path, limites de headers, roteamento até a aplicação, prazos de conexão e requisição, overload manager, stats e admin local | `libn3x.so` |
| Adaptador por alvo (`core/adapter/*`) | Implementa a fábrica de filtro do SDK Go do Envoy; traduz headers, body, resposta local, scheduler e stats para o contrato do SDK N3X; informa as capacidades ausentes no alvo | SDK Go do Envoy (pseudo-versão por alvo) |
| Executor da cadeia (`core/chain`) | Compila a cadeia na criação da config (fases, grupos e recortes); por requisição executa seguir, negar, `Go()` e retomar; resposta em ordem inversa; notifica no fim todas as instâncias que rodaram; 0 alocações no caminho comum | Adaptador, `core/fault` |
| Contenção de falhas (`core/fault`) | `recover` em todo hook e goroutine; políticas fail-open e fail-closed; negação ciente de gRPC; prazos e `max_inflight`; detecção de zumbis; disjuntor por instância (ADR-004) | Executor, telemetria |
| Conexão compartilhada (`core/grpcconn`) | Um `grpc.ClientConn` por processo, com TLS 1.3 e CA montada; backoff, keepalive e prazo máximo por chamada; buffer limitado para envios em segundo plano; modo standalone | Controller, interceptor do plugin core |
| Telemetria (`core/telemetry`) | Métricas `n3x_` como stats do Envoy; logs JSON com campos fixos; amostragem de stack traces | Adaptador |

Quem persiste dados: ninguém no sidecar; o estado é todo volátil. Quem faz cache: só a memória isolada de cada plugin (ADR-005 do `n3x-sdk`). Quem orquestra a requisição: o executor da cadeia. Quem orquestra o processo: o supervisor (ou o `pilot-agent` no modo Istio).

---

### Fluxo de requisições e de dados
**Fluxo de requisição**
- O cliente conecta em `17000` (ou no listener de entrada do Istio)
- O Envoy aplica os limites de headers (60 KiB e no máximo 100), normaliza o path (`normalize_path`, `merge_slashes` e `path_with_escaped_slashes_action: UNESCAPE_AND_REDIRECT`) e gera o `x-request-id` quando ausente; requisição malformada recebe 400 sem chegar à aplicação
- O filtro único do N3X entra no `.so` (cruzamento de request headers)
- O executor identifica o protocolo: gRPC quando `content-type` começa com `application/grpc`, com o método exposto como `/pacote.Serviço/Método`
- O executor percorre as fases `access`, `authn`, `route`, `authz`, `traffic`, `transform`, `upstream` e `observe`. Em cada fase, os grupos rodam em ordem crescente de `order`, e em cada grupo só entram as instâncias cujo recorte (`match`) casa com a requisição; o recorte é avaliado depois da normalização do path
- Cada instância devolve uma ação:
  - **Seguir:** passa para a próxima
  - **Negar:** a cadeia para; em HTTP sai o status do plugin; em gRPC sai resposta trailers-only com HTTP 200, `grpc-status` e `grpc-message` corretos
  - **`Go()`:** o core abre uma goroutine com `recover`, rótulos de pprof (`plugin` e `instance`) e `ctx` com prazo; a requisição fica parada e a cadeia é retomada no worker pelo scheduler do SDK quando a goroutine termina
- **Grupo paralelo** (mesma fase e mesmo `order`, ADR-005): a parte síncrona roda em sequência no worker, em ordem alfabética de `instance`; os trabalhos `Go()` rodam juntos; ninguém vê as mudanças do outro; as mudanças são aplicadas no fim, em ordem alfabética (vale a última); se alguém nega ou falha com fail-closed, vence a negação do primeiro em ordem alfabética e o `ctx` dos demais é cancelado
- Body: só cruza para o Go se alguma instância do recorte declarou hook de body
- O Envoy encaminha para `127.0.0.1:<porta>` mantendo o protocolo, com prazo de conexão de 1 s e de requisição de 15 s. Aplicação fora do ar: 503 ou `UNAVAILABLE`; prazo estourado: 504 ou `DEADLINE_EXCEEDED`
- Na resposta, os grupos rodam em ordem inversa, com as mesmas regras
- No fim da requisição (`on_stream_complete`), todas as instâncias do recorte que implementam o hook são notificadas, inclusive as de fases não alcançadas e quando houve negação; só a instância com o disjuntor aberto não recebe (SID-03). Se o cliente desconectou com trabalho pendente, o `ctx` é cancelado, a requisição é marcada como encerrada e o resultado é descartado sem retomar
- Meta: no máximo 5 cruzamentos Envoy→Go com trabalho da cadeia por requisição, independentemente do número de plugins; as demais entradas do ABI (criação, destruição, pedaços de body e trailers) são passagens O(1) e sem alocação (SID-02)

**Tratamento de falhas por instância** (FR-004)
| Situação | fail-closed | fail-open |
| --- | --- | --- |
| Panic no hook ou na goroutine | 503 / `UNAVAILABLE` | Segue |
| Erro devolvido pelo plugin (`sdk.Fail`) | 503 / `UNAVAILABLE` | Segue |
| Prazo do `Go()` estourado | 504 / `DEADLINE_EXCEEDED` | Segue |
| `max_inflight` atingido (overflow) | 503 / `UNAVAILABLE`, sem abrir goroutine | Segue |
| Disjuntor aberto | 503 / `UNAVAILABLE`, sem chamar o plugin | Pula a instância |

Toda falha incrementa `n3x_plugin_failures_total` com o `reason`; panics também incrementam `n3x_plugin_panics_total`.

**Prazos obrigatórios** (ADR-006): nenhuma operação sem prazo
| Operação | Padrão | Campo no `n3x.yaml` | Ao estourar |
| --- | --- | --- | --- |
| Trabalho assíncrono de plugin | 200 ms | `plugins[].async.timeout` | Política da instância |
| Conexão com a aplicação | 1 s | `app.timeouts.connect` | 503 / `UNAVAILABLE` |
| Requisição à aplicação | 15 s; em gRPC, o `grpc-timeout` do cliente, até 1 h, e sem ele `app.timeouts.request` (SID-06) | `app.timeouts.request` | 504 / `DEADLINE_EXCEEDED` |
| Chamada unária de plugin ao controller | 5 s (teto configurável de 100 ms a 60 s, mesmo com `ctx` maior; o login usa `controller.timeouts.login`; SID-10) | `controller.timeouts.call` | Erro devolvido ao plugin |
| Login no controller | 10 s | `controller.timeouts.login` | Novo login com backoff |
| Health check da aplicação | 1 s, a cada 5 s | `app.health.timeout` e `interval` | Sidecar não pronto |
| `envoy --mode validate` | 10 s | fixo | Container encerra |
| Envoy pronto após subir | 30 s | fixo | Container encerra |
| Drenagem | 20 s | `runtime.drain_timeout` | SIGTERM ao Envoy |
| Encerramento dos plugins (antes do SIGTERM ao Envoy, SID-09) | 2 s | fixo | Encerra mesmo assim; o SIGTERM ao Envoy sai em até 3 s |
| SIGKILL após SIGTERM ao Envoy | 5 s | fixo | SIGKILL |

O SDK só entrega `ctx` com prazo aos plugins, e o `n3x-plugin build` recusa `context.Background()` e `context.TODO()` em código de plugin.

**Fluxo de controle (sempre em segundo plano)** (FR-002)
- Na criação da config do filtro, o core cria o `grpc.ClientConn` sem bloquear, se houver seção `controller`
- O plugin core instala o interceptor do token e faz o login (contrato no HLD do `n3x-plugin-core`, ADR-005 dele: token de 1 h renovado em cerca de 80% do prazo); enquanto o login não completa, as chamadas esperam no buffer ou recebem `UNAVAILABLE`
- O plugin core manda heartbeat a cada `controller.heartbeat_interval` (padrão de 10 s), valor que também vai no registro; o controller marca a instância como desconectada após 3 intervalos sem ping (ADR-007 do `n3x-plugin-core`)
- Reconexão com backoff exponencial de 1 s a 60 s e jitter de 20%; keepalive a cada 30 s com prazo de 10 s
- Envios em segundo plano (eventos e registros) entram num buffer de até 1.000 itens que descarta os mais antigos (`n3x_controller_buffer_dropped_total`)
- Sem seção `controller`: modo standalone; o SDK devolve `ErrStandalone` a quem pedir a conexão e o plugin core fica inativo
- Certificado inválido, credencial recusada ou controller fora do ar nunca afetam o tráfego; só geram log, métrica e nova tentativa

**Fluxo de dados**
- `n3x.yaml` + chave AES (arquivos montados) → entrypoint (valida, decifra para validar, descarta) → bootstrap sem segredos → Envoy
- `n3x.yaml` + chave AES → `.so` (lê, valida e decifra de novo com `core/config`) → cadeia compilada com a config tipada de cada instância
- Manifesto embutido no `.so` → entrypoint (schemas dos plugins e alvo) e plugin core (registro)
- Config efetiva → plugin core (JSON canônico, mascaramento pelo schema, inclusive `controller.secret`, e hash; ADR-006 do `n3x-plugin-core`) → conexão compartilhada → controller
- Eventos do core (`panic`, `breaker_open`, por panics ou por zumbis, `plugin_disabled`, `config_error`, `drain_start` e `secret_in_clear`; lista no ADR-007 do `n3x-plugin-core`) → plugin core → buffer → controller
- Stats `n3x_` → Envoy → `17090` (ou merge do Istio) → Prometheus
- Logs JSON → stderr → coletor do orquestrador

---

### Modelo de dados (alto nível)
Entidades principais
- **`n3x.yaml` v1** do serviço: contrato deste HLD (schema abaixo)
- **Instância de plugin:** `id` (com namespace, padrão do SDK), `instance` (única no serviço), `phase`, `order`, `failure`, `async`, `match` e `config`
- **Cadeia compilada:** fases → grupos (por `order`) → instâncias, com recortes pré-processados (prefixos de path, máscara de métodos e conjunto de serviços gRPC)
- **Manifesto embutido:** plugins, versões, alvo, versão do `core/`, schemas, avisos e hash, em JSON na seção ELF `.n3x_manifest` (formato 1, ADR-003 do `n3x-builder`)
- **Identidade da instância:** serviço + nome do pod ou ID do container
- **Estado do disjuntor por instância:** janela de panics, contagem de zumbis e estado aberto ou fechado
- **Estado por requisição:** fase atual, grupo atual, mudanças pendentes, trabalhos em andamento e marcador de encerrada (reaproveitado via pool, sem alocação)
- **Buffer do controller:** fila limitada de envios em segundo plano

**Schema do `n3x.yaml` v1**
```yaml
version: 1
service: pedidos                     # obrigatório; nome do serviço e da credencial

controller:                          # opcional; ausente = modo standalone
  address: n3x-controller:17443
  ca_file: /etc/n3x/ca.pem
  secret: ENC[v1:...]                # segredo da credencial do serviço "pedidos"
  timeouts: { call: 5s, login: 10s }
  heartbeat_interval: 10s            # intervalo do heartbeat; enviado no registro (ADR-007 do n3x-plugin-core)
  buffer: { max_items: 1000 }        # descarta os mais antigos

keys:
  aes_file: /run/n3x/aes.key         # chave do ENC[...], sempre em arquivo

listener:                            # ignorado no modo Istio
  port: 17000

app:                                 # ignorado no modo Istio, exceto health
  address: 127.0.0.1:3000
  protocol: http1                    # http1 | http2 | grpc
  timeouts: { connect: 1s, request: 15s }
  health:
    type: http                       # http | grpc | tcp
    path: /healthz                   # só para http
    interval: 5s
    timeout: 1s

resources:                           # opcional; padrão = limites do cgroup
  cpus: 2
  memory: 256Mi

runtime:
  drain_timeout: 20s
  log_level: info                    # debug | info | warn | error
  panic_breaker: { panics: 5, window: 60s, zombies: 100 }
  request_body: { max_bytes: 1Mi }   # limite do body para hooks de body (SID-01)
  plugin_limits: {}                  # cotas por plugin: memória, tarefas e conexões (SID-01)

plugins:                             # o plugin core é implícito e não se declara
  - id: n3x.example
    instance: add-header             # única no serviço
    phase: transform                 # access|authn|route|authz|traffic|transform|upstream|observe
    order: 10                        # mesma fase + mesmo order = grupo paralelo
    failure: closed                  # open | closed; padrão = o do manifesto
    async: { timeout: 200ms, max_inflight: 1000 }
    match:                           # recorte; ausente = roda em tudo
      paths: ["/api/"]               # prefixos, avaliados após a normalização
      methods: [GET, POST]
      grpc_services: ["pedidos.v1.Pedidos"]
    config:                          # validada pelo schema do plugin
      name: X-N3X
      value: "1"
```

Regras de validação (no `n3x validate`, no `n3x run` e no `.so`)
- `version` precisa ser `1`; campo desconhecido é erro, para pegar erros de digitação
- `instance` única no serviço; `id` precisa existir no manifesto; `phase` precisa estar entre as fases permitidas pelo manifesto do plugin
- `order` repetido na fase `route` é recusado (ADR-005)
- Valores `ENC[...]` precisam decifrar; o erro cita o caminho do campo, nunca o valor
- Duração e tamanho em formato Go (`200ms`, `20s`) e Kubernetes (`256Mi`)
- Nenhum campo vem de variável de ambiente (FR-001)
- Sem limite de memória (nem cgroup nem `resources.memory`): o sidecar recusa subir (ADR-008)

**Formato `ENC[...]`**
- `ENC[v1:<base64(nonce de 12 bytes || texto cifrado || tag)>]`, com AES-256-GCM
- Dado associado (AAD) igual a `n3x:v1:<service>`, para que o valor cifrado de um serviço não funcione copiado no `n3x.yaml` de outro; por isso o `n3x encrypt` exige `--service` (SID-01)
- Chave: 32 bytes aleatórios em base64, gerados por `n3x keygen`

Relações
- Serviço 1:1 `n3x.yaml` (por versão de deploy)
- `n3x.yaml` 1:N instâncias de plugin; plugin 1:N instâncias
- Fase 1:N grupos; grupo 1:N instâncias
- Instância de plugin 1:1 estado do disjuntor
- `.so` 1:1 manifesto; processo do Envoy 1:1 runtime Go 1:1 conexão compartilhada

Fonte de verdade
- `n3x.yaml` do serviço, montado no container e só alterado por redeploy
- Manifesto embutido no `.so` para a lista de plugins e seus schemas
- O hash da config efetiva reportado ao controller sempre corresponde à config em uso, porque o `.so` só lê o arquivo na subida

Versionamento e retenção
- Schema versionado pelo campo `version`; campos novos só por adição dentro da v1
- Nada é retido no sidecar: estado volátil, perdido no restart

---

### Interfaces públicas
| Nome | Tipo | Protocolo | Exposição | SLAs/Limites |
| ---- | ---- | ---------- | --------- | ------------- |
| Entrada (`17000`) | API | HTTP/1.1, HTTP/2 (h2c) e gRPC | Externa | Latência adicionada p50 ≤ 0,5 ms e p99 ≤ 2 ms a 1.000 rps; headers até 60 KiB e 100 itens |
| Health (`17021`, `/live`, `/ready` e `POST /drain` só de loopback) | API | HTTP, servido pelo entrypoint | Interna (orquestrador; `/drain` só para a aplicação do pod) | Pronto em até 5 s após o container subir; 503 imediato na drenagem |
| Métricas (`17090`, `GET /metrics`) | API | HTTP (Prometheus), listener do Envoy → `/stats/prometheus` | Interna | Só esse caminho e método; nenhum outro endpoint do admin |
| Admin do Envoy (`unix:/tmp/n3x/admin.sock`, modo 0600) | API | HTTP sobre socket Unix | Só o container do sidecar, sem porta TCP | Usado só pelo supervisor, pelo health e pelo listener de métricas; a porta 17001 fica reservada e sem uso (SID-06) |
| CLI `n3x run`, `validate`, `encrypt`, `keygen`, `probe` e `help` (mais `istio-drain` na imagem `istio-*`) | SDK | Processo e stdin/stdout | Operador, CI e health check do orquestrador | `validate` não sobe o Envoy; `encrypt --key --service` lê o segredo do stdin; `probe` consulta a `17021` |
| `n3x.yaml` v1 + chave AES + CA | SDK | Arquivos YAML e binários montados | Orquestrador → container | Lidos só na subida |
| Config do filtro (`config_path`) | API | JSON no bootstrap ou no `EnvoyFilter` | Interna (entrypoint ou Istio → `.so`) | Nunca contém segredos; o caminho da chave vem de `keys.aes_file` do `n3x.yaml` |
| `EnvoyFilter` estático | API | CRD do Istio | Malha | Restrito a `proxyVersion` 1.31 |
| Canal de controle local (`/tmp/n3x/control.sock`) | API | Socket Unix, JSON de uma linha (`ping`, `drain`, `stop`) | Só o entrypoint, conferido por `SO_PEERCRED` | Modo 0600; servido pelo `.so` (SID-09) |
| Conexão compartilhada | API | gRPC com TLS 1.3 até `17443` | Interna (`.so` → controller) | Uma conexão por processo; prazo de até 5 s por chamada; buffer de 1.000 itens |

**Bootstrap gerado (essencial)**
```yaml
admin:
  address: { pipe: { path: /tmp/n3x/admin.sock, mode: 384 } }   # 0600, sem porta TCP (SID-06)
overload_manager:
  resource_monitors:
  - name: envoy.resource_monitors.cgroup_memory
  actions:
  - name: envoy.overload_actions.shrink_heap
    triggers: [{ name: envoy.resource_monitors.cgroup_memory, threshold: { value: 0.90 } }]
  - name: envoy.overload_actions.stop_accepting_connections
    triggers: [{ name: envoy.resource_monitors.cgroup_memory, threshold: { value: 0.95 } }]
static_resources:
  listeners:
  - name: n3x_ingress                          # 0.0.0.0:17000
    # http_connection_manager: codec AUTO, normalize_path, merge_slashes,
    # path_with_escaped_slashes_action UNESCAPE_AND_REDIRECT,
    # max_request_headers_kb 60, headers_with_underscores_action REJECT_REQUEST,
    # request_headers_timeout 10s, generate_request_id true
    # http_filters: [n3x (dynamic module), envoy.filters.http.router]
    # rotas grpc (grpc-timeout do cliente até 1 h, senão app.timeouts.request) e http (timeout = app.timeouts.request) → cluster app (SID-06)
  - name: n3x_metrics                          # 0.0.0.0:17090
    # route exata GET /metrics → cluster admin, prefix_rewrite /stats/prometheus
  clusters:
  - name: app                                  # STATIC 127.0.0.1:<porta>, connect_timeout 1s
    # protocolo por typed_extension_protocol_options (HttpProtocolOptions): HTTP/2 quando app.protocol é http2 ou grpc (SID-06)
  - name: admin                                # STATIC pipe /tmp/n3x/admin.sock
```
Filtro do N3X no bootstrap e no `EnvoyFilter`:
```yaml
name: n3x
typed_config:
  "@type": type.googleapis.com/envoy.extensions.filters.http.dynamic_modules.v3.DynamicModuleFilter
  dynamic_module_config: { name: n3x, do_not_close: true }   # /usr/lib/n3x/libn3x.so pelo search path; o runtime Go não é descarregado
  filter_name: n3x
  filter_config:
    "@type": type.googleapis.com/google.protobuf.StringValue
    value: '{"config_path":"/etc/n3x/n3x.yaml"}'   # sem key_path: a chave vem de keys.aes_file (SID-06)
```
Logs do Envoy em JSON por `application_log_config.log_format.json_format`. O search path de dynamic modules é fixado na imagem em `/usr/lib/n3x`. O bootstrap gerado é comparado com arquivos golden nos testes.

**`EnvoyFilter` estático do modo Istio**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata: { name: n3x-istio-1-31, namespace: istio-system }   # um objeto por alvo (SID-11)
spec:
  workloadSelector: { labels: { n3x.io/enabled: "true" } }
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      proxy: { proxyVersion: '^1\.31\..*' }
      listener:
        filterChain:
          filter:
            name: envoy.filters.network.http_connection_manager
            subFilter: { name: envoy.filters.http.router }
    patch:
      operation: INSERT_BEFORE
      value: { }   # mesmo bloco "Filtro do N3X" acima
```
Igual para toda a malha; o que muda por serviço é só o `n3x.yaml` montado. A normalização de path no modo Istio depende da `meshConfig.pathNormalization` (recomendado `DECODE_AND_MERGE_SLASHES`, mínimo `MERGE_SLASHES`; o `.so` recusa com 400 o path que chegar fora da forma normalizada; SID-11).

**Imagens de runtime**
- `ghcr.io/giustech/n3x/sidecar:<semver>-envoy-1.40` a partir de `envoyproxy/envoy:v1.40.x@sha256:...`; `ghcr.io/giustech/n3x/sidecar:<semver>-istio-1.31` a partir de `istio/proxyv2:1.31.x@sha256:...`; amd64 e arm64, formato OCI, com semver próprio (ADR-001 do `n3x-builder`)
- Conteúdo: `/usr/local/bin/n3x`, `/etc/n3x/target`, diretório `/usr/lib/n3x/` vazio para o `.so` copiado no estágio final do Dockerfile do operador, `ENTRYPOINT ["/usr/local/bin/n3x"]` com `CMD ["run"]` (para `docker run <imagem> encrypt ...` e o despacho do modo Istio funcionarem; SID-08)
- A imagem não traz `.so` padrão; ele sempre sai do `n3x-plugin build` (ADR-001)

---

### Considerações de escalabilidade e disponibilidade
Abordagem geral
- O sidecar escala 1:1 com a aplicação, sem estado compartilhado entre réplicas
- O controller nunca está no caminho da requisição: zero erros da aplicação causados por ele
- Proteção em camadas para um plugin ruim não derrubar o sidecar: prazos, `max_inflight`, disjuntor, `GOMEMLIMIT` e overload manager
- Erro fatal do Go derruba o processo; a recuperação é o restart pelo orquestrador e a existência de mais de uma réplica por serviço

Técnicas aplicadas
- **Runtime calculado pelo cgroup** (ADR-008):
  - `CPU` = `resources.cpus` ou o limite do cgroup (`cpu.max`), arredondado para cima, mínimo 1
  - `MEM` = `resources.memory` ou o limite do cgroup (`memory.max`); sem nenhum dos dois, o sidecar recusa subir
  - `envoy --concurrency` = `CPU`
  - `GOMAXPROCS` = `max(CPU, 2)`, sempre maior ou igual aos workers
  - `GOMEMLIMIT` = 45% de `MEM`
  - Overload manager com o monitor `cgroup_memory`: reduz o heap em 90% e para de aceitar conexões em 95%
  - Exemplo com 1 vCPU e 256 MiB: 1 worker, `GOMAXPROCS=2`, `GOMEMLIMIT` de cerca de 115 MiB
- **Backpressure:** até 1.000 trabalhos `Go()` simultâneos por instância (os zumbis contam até terminarem de fato); buffer de 1.000 itens para o controller
- **Disjuntor por instância** (ADR-004): abre com 5 panics em 60 s ou com mais de 100 zumbis; aberto, aplica a política sem chamar o plugin; fecha só no restart; gera evento e métrica
- **Recortes sem custo:** instância fora do recorte não é chamada; o teste custa uma comparação de prefixo, método ou serviço
- **Drenagem de 20 s** com sidecar nativo no Kubernetes; zero requisições cortadas dentro do prazo
- **Reconexão com jitter** ao controller, evitando rajadas

**Orçamento de desempenho do sidecar** (ADR-009), com carga open-loop a 1.000 rps abaixo do joelho da curva, plugin core e plugin de exemplo ativos, medido contra o Envoy puro na mesma máquina
| Parcela | p50 | p99 |
| --- | --- | --- |
| Envoy puro (proxy sem o filtro N3X) | 0,30 ms | 1,00 ms |
| Core do `.so` (adaptador + executor, até 5 cruzamentos com trabalho da cadeia) | 0,15 ms | 0,50 ms |
| Plugins síncronos (core + exemplo) | 0,05 ms | 0,10 ms |
| Folga | | 0,40 ms |
| **Total (meta do PRD)** | **≤ 0,5 ms** | **≤ 2 ms** |

| Recurso ocioso | Envoy | Runtime Go + core | Plugins | Entrypoint (supervisor e health) | Total (meta) |
| --- | --- | --- | --- | --- | --- |
| Memória | 45 MiB | 35 MiB | 20 MiB | 20 MiB (hipótese) | ≤ 100 MiB |
| CPU | 0,02 vCPU | 0,02 vCPU | 0,01 vCPU | 0,005 vCPU (hipótese) | ≤ 0,05 vCPU |

As parcelas somam mais que a meta (120 MiB e 0,055 vCPU): a meta dura é o total do container, e as parcelas são rebalanceadas pelo benchmark (regra S5 da SID-12) antes da v1.

- 0 alocações por requisição no executor, nos recortes e na negação (`-benchmem` como portão da CI)
- Dimensionamento recomendado: 1 vCPU e 256 MiB para até 1.000 rps (hipótese, confirmada pelo benchmark)

Meta de disponibilidade
- Zero requisições da aplicação com erro causado pelo controller
- Zero requisições cortadas em rolling update, dentro do prazo de drenagem
- Sidecar pronto em até 5 s após o container subir
- Toda requisição termina: panic contido, prazos em toda operação e disjuntor por instância
- O sidecar não tem meta própria de uptime além da aplicação: é parte da mesma unidade de deploy

---

### Segurança
Autenticação
- Até o controller: TLS 1.3 com verificação do certificado pela CA em `controller.ca_file`; credencial do serviço (`service` + `controller.secret` em `ENC[...]`) usada só no login; token de sessão HMAC guardado só na memória privada do plugin core (ADR-003 e ADR-004 do `n3x-sdk`)
- A autenticação do tráfego da aplicação não é papel do sidecar na v1; fica para plugins futuros

Autorização
- Só o plugin core instala o interceptor na conexão compartilhada; os demais plugins nunca enxergam o token
- Só as fases até `route` podem alterar a rota (contrato do SDK); o executor não oferece essa API às fases seguintes
- A negação gRPC sempre escreve `grpc-status` e `grpc-message`; resposta local crua é proibida aos plugins

Proteção de dados
- Segredos decifrados nunca vão para o bootstrap, o `/config_dump`, logs, mensagens de erro ou a config enviada ao controller; o `.so` decifra sozinho a partir dos arquivos (ADR-007)
- Buffers de segredos zerados após o uso, por melhor esforço
- Endurecimento do processo e da imagem:
  - Usuário não root (UID e GID `17000`; `1337` no modo Istio)
  - Sistema de arquivos raiz somente leitura, com escrita só em `/tmp` (tmpfs)
  - Nenhuma capability do Linux (`drop: ALL`)
  - `.so` carregado só de `/usr/lib/n3x/libn3x.so`, dentro da imagem, nunca de fonte remota, com alvo e integridade conferidos contra o manifesto
  - Chave AES com permissão mais aberta que `0440` gera aviso, sem impedir a subida
- Normalização de path antes dos recortes, para que `/api/../admin` não escape de um recorte
- Limites de headers (60 KiB e 100 itens) e rejeição de headers com underscore
- Aplicação nunca exposta: fora do Istio ela escuta só em `127.0.0.1`
- O sidecar não trata PII de negócio; os logs não registram headers, corpos nem query strings

Gestão de segredos
- `ENC[v1:...]` com AES-256-GCM e AAD ligado ao serviço; chave de 32 bytes em arquivo montado, nunca em variável de ambiente
- `n3x keygen` gera a chave e `n3x encrypt --key <arquivo> --service <serviço>` cifra o segredo lido do stdin (resolve a pendência do HLD de plataforma)
- Rotação da chave AES: gerar chave nova, recifrar os valores e redeployar; a v1 não aceita duas chaves ao mesmo tempo

---

### Observabilidade
Logs
- JSON em stderr, uma linha por evento, com `ts`, `level`, `component` (`entrypoint`, `envoy`, `core` ou `plugin`), `service`, `instance`, `plugin_id` e `request_id` quando houver
- Entrypoint: subida, validação, valores efetivos de runtime (`concurrency`, `GOMAXPROCS` e `GOMEMLIMIT`), health e drenagem
- Envoy: logs da aplicação do Envoy em JSON
- Core e plugins: via SDK; stack trace de panic amostrado em no máximo 1 por segundo por instância
- Nunca registra segredos, tokens, headers, corpos ou query strings

Métricas (stats do Envoy definidas pelo `.so`, expostas em `17090` ou pelo merge do Istio)
| Métrica | Tipo | Rótulos |
| --- | --- | --- |
| `n3x_controller_connected` | gauge 0/1 | |
| `n3x_controller_session_age_seconds` | gauge | |
| `n3x_controller_logins_total` | counter | `result` |
| `n3x_controller_buffer_items` | gauge | |
| `n3x_controller_buffer_dropped_total` | counter | |
| `n3x_plugin_panics_total` | counter | `plugin`, `instance` |
| `n3x_plugin_breaker_open` | gauge 0/1 | `plugin`, `instance` |
| `n3x_plugin_failures_total` | counter | `plugin`, `instance`, `reason` (`timeout`, `error`, `panic`, `overflow`, `body_limit` ou `breaker`) |
| `n3x_plugin_denies_total` | counter | `plugin`, `instance`, `code` |
| `n3x_plugin_async_inflight` | gauge | `plugin`, `instance` |
| `n3x_plugin_async_zombies` | gauge | `plugin`, `instance` |
| `n3x_plugin_duration_seconds` | histogram (só `_bucket` e `_count` no sidecar) | `plugin`, `instance`, `phase`, `hook` |
| `n3x_plugin_memory_bytes` | gauge | `plugin` |
| `n3x_build_info` | gauge = 1 | `version`, `target`, `config_hash` |

- Nenhum rótulo com path, header ou valor vindo do usuário
- Catálogo completo, com as famílias `n3x_sidecar_*`, `n3x_config_secrets_in_clear` e as de memória, tarefas e conexões por plugin: SID-02, contrato 9 (e os acréscimos pedidos pelas SID-04, SID-05, SID-07, SID-08, SID-10, SID-11 e SID-12)
- As stats padrão do Envoy (upstream, conexões e overload) continuam disponíveis em `/metrics`
- A observabilidade do tráfego da aplicação continua fora da v1; `n3x_plugin_duration_seconds` mede o custo do N3X

Tracing
- Sem tracing distribuído na v1 (ADR-009 do `n3x-sdk`)
- Correlação por `request_id` nos logs do sidecar e nos metadados gRPC até o controller
- Goroutines de plugin rotuladas por pprof (`plugin` e `instance`) para investigar zumbis e vazamentos no perfil de goroutines

Dashboards e alertas
- Alertas sugeridos ao operador:
  - `n3x_plugin_breaker_open == 1` (disjuntor aberto)
  - `rate(n3x_plugin_panics_total[5m]) > 0`
  - `n3x_plugin_async_zombies > 0`
  - `n3x_controller_connected == 0` por mais de 5 minutos
  - `rate(n3x_controller_buffer_dropped_total[5m]) > 0`
  - `rate(n3x_plugin_failures_total{reason="overflow"}[5m]) > 0` (backpressure atuando)
- Painel de referência (hipótese, em `n3x-sdk/deploy/`): latência p50 e p99 por plugin, falhas por motivo, disjuntores, zumbis, conexão com o controller e uso de memória em relação a `GOMEMLIMIT`
- Health em `17021` usado pelo orquestrador; os eventos de disjuntor, zumbis e drenagem também aparecem no painel do controller via plugin core

---

### Riscos arquiteturais e mitigação
#### Plugin ignora o `ctx` e a goroutine continua viva depois do prazo
- **Probabilidade:** média (era alta antes das mitigações; é o risco mais crítico do sidecar)
- **Impacto:** goroutines e memória acumulam até derrubar o sidecar; Go não consegue matar uma goroutine por fora
- **Mitigação:**
  - Análise estática no `n3x-plugin build`: recusa `time.Sleep`, `http.Get`, `net.Dial`, `db.Query` sem contexto, recebimento de canal sem `select` com `ctx.Done()`, laço sem checagem do `ctx` dentro de `Go()` e `context.Background()` ou `context.TODO()` (regra definida aqui, implementada no builder)
  - Ferramentas do SDK com prazo próprio: a conexão compartilhada impõe `min(prazo do ctx, 5 s)` num interceptor do core; clientes HTTP e TCP do SDK com timeout no transporte; `sdk.Sleep(ctx, d)`
  - Detecção de zumbis com rótulos de pprof, métrica `n3x_plugin_async_zombies`, log amostrado e alerta
  - Zumbis contam no `max_inflight` até terminarem e mais de 100 zumbis abrem o disjuntor da instância
  - Teste obrigatório `sdktest.AssertHonorsCancel` no kit do SDK, com o plugin de exemplo como modelo
- **Plano de contingência:** um laço de CPU que ignora o `ctx` só para com o restart; ele fica isolado pelo disjuntor e visível no painel até a instância sair do `n3x.yaml` ou o plugin ser corrigido, com redeploy

#### Erro fatal do Go ou crash em C dentro do `.so` derruba o Envoy
- **Probabilidade:** média
- **Impacto:** o serviço perde a entrada até o container reiniciar
- **Mitigação:**
  - `recover` em todo hook e em toda goroutine criada pelo core
  - Instâncias de plugin imutáveis e estado por requisição
  - Análise estática proibindo cgo, `unsafe` e `reflect` sobre tipos do SDK
  - `GOMEMLIMIT` e overload manager contra OOM
  - Testes com `-race`, fuzzing e longa duração com core dump
- **Plano de contingência:** o orquestrador reinicia o container; o plugin problemático sai do `n3x.yaml` ou da imagem; mais de uma réplica por serviço reduz o efeito

#### Bug na iteração de filtros (seguir, parar, retomar e grupos) prende ou corrompe a requisição
- **Probabilidade:** média
- **Impacto:** requisições presas até o timeout ou mudanças de header não aplicadas
- **Mitigação:**
  - TDD no executor
  - Casos golden em HTTP/1.1, HTTP/2 e gRPC
  - Testes de propriedade sobre seguir, parar, retomar e grupos paralelos
  - Prazos em toda operação, para que nada fique preso sem fim
- **Plano de contingência:** os prazos do Envoy encerram a requisição e a instância envolvida sai do `n3x.yaml`, com redeploy

#### Negação gRPC chega ao cliente como sucesso (`grpc-status 0`)
- **Probabilidade:** alta
- **Impacto:** requisição negada aparece como bem-sucedida, o que é uma falha de segurança
- **Mitigação:**
  - Negação do SDK sempre em trailers-only com `grpc-status` e `grpc-message`
  - Resposta local crua proibida aos plugins
  - Teste de integração obrigatório nos alvos `envoy-1.40` e `istio-1.31`
- **Plano de contingência:** a release é bloqueada se o teste falhar

#### Overhead acima do orçamento
- **Probabilidade:** média
- **Impacto:** p99 acima de 2 ms
- **Mitigação:**
  - Benchmark open-loop contra o Envoy puro como portão noturno
  - 0 alocações no caminho comum e até 5 cruzamentos com trabalho da cadeia por requisição
  - Body só cruza para o Go quando alguma instância pede
- **Plano de contingência:** otimizar os caminhos quentes ou rever o orçamento com base nas medições antes de fechar a v1

#### Retomada depois que a requisição já foi destruída
- **Probabilidade:** média
- **Impacto:** uso de memória já liberada pelo Envoy, com crash do processo
- **Mitigação:**
  - Estado por requisição com marcador de encerrada, conferido antes de qualquer retomada pelo scheduler
  - Cancelamento do `ctx` no `on_stream_complete`
  - Fuzzing com desconexão do cliente durante trabalho assíncrono
- **Plano de contingência:** restart pelo orquestrador; o caso que reproduz o problema vira teste de regressão

#### Vazamento de goroutines ou memória na conexão compartilhada
- **Probabilidade:** média
- **Impacto:** aumento de latência e de memória no sidecar
- **Mitigação:**
  - Uma conexão por processo e buffer limitado com descarte dos mais antigos
  - Teste de longa duração com o perfil de goroutines
  - `GOMEMLIMIT`
- **Plano de contingência:** rodar o serviço em modo standalone (sem seção `controller`) até a correção

#### Fórmula de `GOMEMLIMIT` errada causa OOM ou GC excessivo
- **Probabilidade:** média
- **Impacto:** restart por OOM ou latência alta por GC frequente
- **Mitigação:**
  - Teste de carga com limite de memória apertado
  - Valores efetivos no log e uso de memória em relação ao limite no painel de referência
- **Plano de contingência:** ajustar `resources.memory` no `n3x.yaml` e revisar a fórmula numa versão nova do sidecar

#### Modo Istio: patch do Istio ignora as variáveis de runtime ou o `EnvoyFilter`
- **Probabilidade:** alta
- **Impacto:** o N3X não carrega, roda sem os ajustes do runtime Go ou derruba o proxy
- **Mitigação:**
  - CI por patch do Istio, com teste de carga do `.so` na imagem exata
  - `EnvoyFilter` restrito a `proxyVersion` 1.31
  - O `.so` loga os valores de runtime que realmente recebeu
- **Plano de contingência:** fixar o patch suportado no catálogo e, naquele serviço, rodar o sidecar do N3X sem Istio

#### Aplicação alcança o admin do Envoy
- **Probabilidade:** baixa (era média com o admin em `127.0.0.1:17001`)
- **Impacto:** como a aplicação divide o namespace de rede com o sidecar, um processo comprometido nela poderia chamar `/quitquitquit`, `/drain_listeners` ou `/config_dump`
- **Mitigação:**
  - Admin só no socket Unix `/tmp/n3x/admin.sock` (0600) dentro de `/tmp/n3x` (0700), sem porta TCP (decidido em 2026-10-22; SID-06)
  - Nenhum segredo no `/config_dump` (ADR-007)
  - Guia: o `/tmp` do sidecar nunca é volume compartilhado com a aplicação
- **Plano de contingência:** se um orquestrador exigir `/tmp` compartilhado, mover o socket para um diretório próprio numa minor

#### Grupo paralelo com resultado diferente entre execuções
- **Probabilidade:** baixa
- **Impacto:** comportamento não reproduzível
- **Mitigação:**
  - Mudanças aplicadas no fim do grupo, em ordem alfabética de `instance`
  - Negação escolhida pela mesma ordem
  - Testes determinísticos com o agendador do kit do SDK
- **Plano de contingência:** separar as instâncias em `order` distintos no `n3x.yaml`

---

### ADRs e próximos passos
ADRs associados
- [ADR-001: Core do `.so` como biblioteca, com o `.so` sempre gerado pelo `n3x-plugin build`](adr/ADR-001-core-como-biblioteca.md)
- [ADR-002: Health `17021` no entrypoint e métricas `17090` no Envoy](adr/ADR-002-health-no-entrypoint-metricas-no-envoy.md)
- [ADR-003: Subcomandos `run`, `validate`, `encrypt` e `keygen` no entrypoint](adr/ADR-003-subcomandos-do-entrypoint.md)
- [ADR-004: Disjuntor por instância de plugin, por panics e por zumbis](adr/ADR-004-disjuntor-por-instancia.md)
- [ADR-005: Grupo paralelo por fase e `order`](adr/ADR-005-grupo-paralelo-por-order.md)
- [ADR-006: Prazo obrigatório em toda operação](adr/ADR-006-prazo-obrigatorio.md)
- [ADR-007: O `.so` lê o próprio `n3x.yaml`, sem segredos no bootstrap](adr/ADR-007-so-le-o-proprio-n3x-yaml.md)
- [ADR-008: Runtime calculado pelo cgroup e recusa sem limite de memória](adr/ADR-008-runtime-pelo-cgroup.md)
- [ADR-009: Orçamento de latência e recursos do sidecar](adr/ADR-009-orcamento-do-sidecar.md)
- Herdados do `n3x-sdk`: ADR-003 (login e sessão HMAC), ADR-004 (capacidades do plugin core), ADR-005 (memória isolada), ADR-006 (portas 17xxx e Istio no comando), ADR-007 (sidecar nativo no Kubernetes), ADR-009 (sem tracing) e ADR-010 (metas no nível do sistema)

Decisões pendentes
- Resolvida em 2026-10-22: admin do Envoy em socket Unix (`/tmp/n3x/admin.sock`), sem porta TCP (SID-06; PRD atualizado). Pendência externa: refletir no ADR-006 do `n3x-sdk`
- Resolvida: comando de health dentro do container para ECS e docker compose, com o subcomando `n3x probe live|ready` (SID-09)
- Resolvida: canal de controle local pelo socket Unix `/tmp/n3x/control.sock`, servido pelo `.so` (SID-09); no modo Istio, a drenagem é detectada pelo admin do Envoy, com o preStop `n3x istio-drain` como contingência (SID-11)
- Resolvida: detecção do modo Istio pelo primeiro argumento `proxy`, com repasse dos demais subcomandos do `pilot-agent` (SID-11)
- Regras da análise estática de `ctx` e de bloqueio: implementação no HLD do `n3x-builder`
- `sdk.Sleep`, clientes com prazo, `AssertHonorsCancel` e cota padrão de memória por plugin: HLD do `n3x-sdk`
- Resolvida: formato da config efetiva, mascaramento e hash (ADR-006 do `n3x-plugin-core`)
- Pseudo-versão do SDK Go do Envoy e patch exato do Istio suportado: catálogo do `n3x-builder`
- Ajuste das fórmulas de `GOMEMLIMIT` e do orçamento após o primeiro benchmark: SID-12 (regras S1 a S7)
- Resolvida em 2026-10-22: atualização do PRD no FR-001 (credencial como `service` do topo + `controller.secret`) e no FR-004 (grupo paralelo por `order` e disjuntor por instância, também por zumbis)

Próximos passos
- Implementar `core/config` com o schema v1 e testes de tabela e fuzzing, base para o `n3x validate`
- Implementar o executor (`core/chain`) e a contenção de falhas (`core/fault`) com TDD e benchmark de 0 alocações
- Gerar o bootstrap com arquivos golden e validar com `envoy --mode validate` nas duas imagens
- Prova de carga do `.so` com o plugin de exemplo no `istio/proxyv2:1.31.x`, para confirmar os símbolos antes de declarar o alvo suportado
- Montar o benchmark open-loop contra o Envoy puro e confirmar o orçamento
- Escrever o FDD/LLD do executor da cadeia e da conexão compartilhada
- Publicar o `EnvoyFilter` e os exemplos de `n3x.yaml` em `n3x-sdk/deploy/`
