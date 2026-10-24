# DRAFT_CONSTITUITION: n3x-sidecar

> Arquivo preparatório para o `/speckit-constitution`. Não é a constituição: descreve o que o repositório é,
> o que faz, para que serve, quem usa e onde usa. Versão curada em 2026-10-24, sem lacunas abertas.

## 1. O que é

O `n3x-sidecar` é um dos cinco repositórios da plataforma N3X (Network Navigation Node eXtensible), um
middleware de rede extensível que funciona como camada de override sobre um proxy Envoy em modelo sidecar.
O repositório entrega duas coisas (ADR-001):

- o binário `n3x` (`cmd/n3x`), entrypoint supervisor em Go embutido na imagem de runtime
  `ghcr.io/giustech/n3x/sidecar:<versão>-<alvo>` (alvos `envoy-1.40` e `istio-1.31`, amd64 e arm64);
- o módulo Go `core/`, biblioteca importada pelo `n3x-plugin build` do `n3x-builder`, que gera o único
  `libn3x.so` (dynamic module do Envoy). O repositório nunca entrega `.so` de produção: só compila `.so`
  de desenvolvimento (`cmd/libn3x-dev`, tag `n3x_dev`) e de teste.

Módulos Go: raiz `github.com/giustech/n3x-sidecar` (`cmd/`, `internal/`, `images/tools/`), módulo aninhado
`github.com/giustech/n3x-sidecar/core` com `go.mod` próprio e tag `core/vX.Y.Z`, e `bench/` como módulo
próprio (hipótese H-01 da SID-12). Licença Apache 2.0.

## 2. O que faz

- Lê e valida o `n3x.yaml` v1 do serviço com uma única função (`config.Load`), usada no `validate`, no `run`
  e no `.so`; nenhum comportamento vem de variável de ambiente do operador
- Confere o `.so` antes de subir (alvo, integridade, versão do `core/`) e detecta a identidade da instância
- Gera o bootstrap do Envoy sem segredos, valida com `envoy --mode validate` e supervisiona o Envoy como PID 1
- Calcula `--concurrency`, `GOMAXPROCS`, `GOMEMLIMIT` e `GODEBUG` a partir do cgroup (ADR-008)
- Executa, num único filtro, a cadeia de plugins em oito fases fixas (`access`, `authn`, `route`, `authz`,
  `traffic`, `transform`, `upstream`, `observe`), com grupos paralelos por `order` (ADR-005), delegando as
  decisões de cadeia ao motor `chain` do `n3x-sdk`
- Contém a falha de cada instância de plugin: fail-open/fail-closed, negação ciente de gRPC, prazos
  (ADR-006), `max_inflight`, zumbis e disjuntor por instância (ADR-004)
- Mantém uma única conexão gRPC (TLS 1.3) com o controller, sempre fora do caminho da requisição
- No modo padrão, serve health (`17021`), expõe métricas (`17090`) e drena as conexões no SIGTERM
- No modo Istio, valida o mesmo `n3x.yaml`, aplica o runtime Go e faz `exec` do `pilot-agent`
- CLI: `run`, `validate`, `encrypt`, `keygen`, `probe`, `help`; na imagem `istio-*`, `proxy` e `istio-drain`

## 3. Para que serve

Tira os comportamentos de rede (rate limit, forward, redirects, observabilidade) do código de cada aplicação
e de cada linguagem e os move para plugins Go dentro do Envoy. Mudar um comportamento é editar o `n3x.yaml`
do serviço e redeployar, sem tocar no código da aplicação. Na v1 nenhum plugin de tráfego é entregue além do
plugin de exemplo do SDK: o objetivo é entregar a base da plataforma.

## 4. Quem usa

- Time de plataforma/SRE: escreve o `n3x.yaml`, gera a chave AES, cifra segredos e opera os sidecars
- Times de produto: colocam o sidecar ao lado da aplicação, sem alterar o código dela
- Autores de plugin (na v1, só o responsável pelo projeto): consomem o `core/` indiretamente pelo SDK
- `n3x-builder`: importa o `core/` e gera o `libn3x.so`; a imagem de build é 1:1 com a versão do `core/`
- Mantenedor: uma pessoa só (Guilherme Giusti) mantém os cinco repositórios

## 5. Onde é usado

Onde houver containers, em cloud ou on-premises, sempre no mesmo namespace de rede da aplicação:

- Kubernetes 1.29 ou superior, como sidecar nativo (`initContainers` com `restartPolicy: Always`)
- ECS em EC2 e em Fargate (metadata v4), na mesma task (`awsvpc`), com health check por `n3x probe live`
- docker compose, com a aplicação em `network_mode: "service:<sidecar>"`
- Istio 1.31, só como compatibilidade e condicionado ao portão G0 do spike (SID-11): imagem `istio-1.31` no
  lugar do `proxyv2` e `EnvoyFilter` estático

## 6. Fronteiras

- Não é dono dos contratos de plugin, das fases, do motor `chain` (inclusive o `recover` dos hooks
  síncronos e a tabela de efeitos) nem do pacote `secret` (`n3x-sdk`)
- Não é dono de login, sessão, registro, heartbeat nem da config efetiva (`n3x-plugin-core`)
- Não gera o `.so`, não aplica a análise estática nem mantém o catálogo de alvos (`n3x-builder`)
- Fora da v1: tráfego de saída, iptables, injeção automática, tracing e observabilidade do tráfego da app

## 7. Stack e restrições técnicas

- Go 1.26 ou superior; adaptadores por build tag (`n3x_envoy140`, `n3x_istio131`), exatamente uma por build
- Envoy 1.40 (`envoyproxy/envoy`) e istio-proxy 1.31 (`istio/proxyv2`), imagens fixadas por digest
- SDK Go de dynamic modules do Envoy em pseudo-versão; só `core/adapter/*` com build tag o importa
- YAML com `go.yaml.in/yaml/v3` (via `yamlconv` do `n3x-sdk`); `log/slog` em JSON; stats nativas do Envoy
- Binário `n3x` estático (`CGO_ENABLED=0`), sem grpc-go, sem SDK do Envoy e sem `go-control-plane`,
  conferido por `go list -deps`; grpc-go só no `.so`, em `core/grpcconn`
- Pacotes do entrypoint só com a biblioteca padrão, `golang.org/x/term`, `golang.org/x/sys/unix` e pacotes
  do `n3x-sdk`; `core/` e plugins usam a mesma versão do `n3x-sdk`
- AES-256-GCM para `ENC[v1:...]`, com AAD `n3x:v1:<service>` e chave em arquivo montado

## 8. Candidatos a princípios (observados nos documentos)

- Toda requisição termina; o controller nunca está no caminho da requisição nem afeta o tráfego
- Nenhum segredo em bootstrap, `/config_dump`, log, mensagem de erro, métrica ou config enviada (canário)
- Prazo obrigatório em toda operação e `ctx` sempre com prazo (ADR-006); só os streams não têm teto total
- Admin do Envoy só em socket Unix, sem porta TCP; a aplicação nunca fica exposta, só o sidecar
- Config estática: só muda por redeploy; mesmo veredito no `validate`, no `run` e no `.so`
- Sem fallback silencioso: valor inválido nunca vira padrão, recusa explícita em vez de bypass
- 0 alocações no caminho da requisição; até 5 entradas do Envoy no Go com trabalho da cadeia
- Orçamento do modo padrão: p50 ≤ 0,5 ms e p99 ≤ 2 ms a 1.000 rps; ocioso ≤ 100 MiB e 0,05 vCPU no
  container inteiro; pronto em ≤ 5 s; sem limite de memória o sidecar não sobe (mínimo 128 MiB)
- Negação gRPC nunca sai como `grpc-status 0`; erro gerado pelo Envoy nunca sai como `OK`
- Sem `init()` com efeito, sem `os.Exit` em biblioteca, sem `os/exec` depois de iniciar o Envoy
- Contratos públicos (códigos de saída, `N3X-Sxxx`, `reason`, métricas, logs) estáveis na major
- Agnóstico de orquestrador, sem recurso exclusivo de nenhum

## 9. Qualidade, testes e portões

- TDD obrigatório em todo código de produção de `cmd/`, `internal/` e `core/`: teste antes do código
- `go test -race`, fuzzing dos parsers, benchmarks `-benchmem` com 0 alocações como portão
- Golden (bootstrap, logs, `main` gerado) e integração com o Envoy real em cada alvo e arquitetura
- Benchmark open-loop contra o Envoy puro (wrk2, Nighthawk ou ghz com HdrHistogram), à noite, na máquina
  dedicada `n3x-perf`; release só com o benchmark verde
- CI no GitHub Actions (`ubuntu-24.04`, `ubuntu-24.04-arm` e runner `self-hosted, n3x-perf`)
- Cadência: no PR, unidade, race, golden, lint e fumaça amd64 (mais o IST-03 quando o PR mexe em
  `core/adapter`); à noite e na pré-release, arm64, Istio, benchmark e longa duração; release só com tudo verde
- Lint em todo PR e no repositório inteiro: `gofmt`, `go vet`, `staticcheck` e `gosec`; `govulncheck` como
  relatório no PR e bloqueando a release
- Cobertura como portão da CI, por pacote: 90% na maioria, 85% em `internal/supervisor`,
  `internal/preflight`, `internal/health` e `core/grpcconn`, 80% nos adaptadores com ponte falsa

## 10. Fluxo de desenvolvimento e governança

- Specs em `specs/NN_nome`, uma por FDD, em ordem de dependência (01 a 12; a 12 é condicional ao portão G0)
- Hipóteses marcadas "(hipótese)" e validadas no spike da fase 0, dentro do `/speckit-plan` de cada spec
- Documentação em pt-BR, exceto o `README.md`, em inglês; identificadores, campos e comandos em inglês
- Decisões registradas inline e em blocos "Clarificações"; ADRs revistos por "Nota de revisão"
- Precedência: PRD, HLD e ADRs prevalecem sobre as FDDs; a FDD que diverge é revista antes da spec dela
- Versionamento: semver do `core/` igual à versão da imagem; `.so` aceito com a mesma major e minor igual ou
  menor; mudança de contrato só por adição na major
- Trunk-based: branches curtas a partir de `main`, merge só por PR com a CI verde, commits em Conventional
  Commits; tags `vX.Y.Z` e `core/vX.Y.Z` no mesmo commit

## 11. Decisões da consolidação (2026-10-24)

- Lacunas: TDD em todo código de produção; lint único; cobertura por pacote como portão; trunk-based com
  Conventional Commits
- Precedência geral: PRD, HLD e ADRs sobre as FDDs. Com isso, valem o 400 para requisição malformada, as
  ferramentas de benchmark do PRD e a memória mínima de 128 MiB também no `validate` (nota do ADR-008)
- Exceções a favor das FDDs: versão da imagem igual à do `core/`, com a regra da série 0 (SID-05, SID-08 e
  SID-11); modo Istio com regras próprias da SID-11 (512 MiB, health e admin do Istio, fora do benchmark);
  cadência de CI das FDDs
- PRD, HLD, ADRs e FDDs não foram alterados; o alinhamento dos textos fica para a convenção de Clarificações

## 12. Fontes

`docs/prd/PRD-N3X.md` (v1.5), `docs/HLD.md` (v1.1), `docs/adr/ADR-001` a `ADR-009`, `docs/fdd/FDD-SID-01`
a `SID-12`, `specs/README.md`.
