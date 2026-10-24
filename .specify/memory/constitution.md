<!--
Sync Impact Report
- Mudança de versão: template sem versão → 1.0.0 (primeira ratificação)
- Princípios (todos os placeholders substituídos):
  - [PRINCIPLE_1_NAME] → I. O tráfego nunca depende do controller e toda requisição termina
  - [PRINCIPLE_2_NAME] → II. Nenhum segredo sai do processo
  - [PRINCIPLE_3_NAME] → III. Superfície mínima e processo endurecido
  - [PRINCIPLE_4_NAME] → IV. Configuração estática, fonte única e veredito único
  - [PRINCIPLE_5_NAME] → V. Prazo obrigatório e recursos limitados
  - Acrescentados: VI. Desempenho é contrato medido; VII. Fronteiras de módulo e dependência
    verificadas; VIII. Test-First (NÃO NEGOCIÁVEL); IX. Compatibilidade por adição
- Seções acrescentadas: "Restrições técnicas e de plataforma" ([SECTION_2_NAME]) e
  "Fluxo de desenvolvimento e portões de qualidade" ([SECTION_3_NAME])
- Seções removidas: nenhuma
- Templates e documentos:
  - ✅ .specify/templates/tasks-template.md: testes deixam de ser opcionais (Princípio VIII)
  - ✅ .specify/templates/plan-template.md: sem alteração; o "Constitution Check" é preenchido
    a partir desta constituição pelo /speckit-plan
  - ✅ .specify/templates/spec-template.md: sem alteração
  - ✅ .specify/templates/checklist-template.md: sem alteração
  - ✅ README.md: link para esta constituição
  - ⚠ .claude/skills/speckit-tasks/SKILL.md: diz que testes são opcionais salvo pedido de TDD;
    não editado por ser arquivo gerenciado pela integração do Spec Kit. Esta constituição é o
    pedido de TDD e prevalece
- TODOs de acompanhamento (não são placeholders):
  - Rever, antes da spec de cada uma, as FDDs que divergem do PRD, do HLD ou dos ADRs:
    SID-01 (faixa de memória), SID-04 e SID-10 (tarefa de fundo sem prazo), SID-06 (status
    de requisição malformada), SID-09 (porta 17021 e dono da drenagem) e SID-12 (ferramenta
    de carga)
-->

# Constituição do n3x-sidecar

## Princípios fundamentais

### I. O tráfego nunca depende do controller e toda requisição termina

- O controller NUNCA está no caminho da requisição. A conexão gRPC com ele é única por processo,
  usa TLS 1.3 e só é usada por tarefas de fundo. Controller fora do ar, certificado inválido ou
  credencial recusada DEVEM gerar só log, métrica e nova tentativa, sem afetar o tráfego. Sem a
  seção `controller` no `n3x.yaml`, o sidecar roda em modo standalone.
- Toda requisição DEVE terminar. Panic, erro, prazo estourado ou `max_inflight` atingido num plugin
  são contidos por instância e seguem a política `failure` dela: `open` segue a cadeia; `closed`
  responde `503`/`UNAVAILABLE`, ou `504`/`DEADLINE_EXCEEDED` quando o prazo estoura. O `500`/
  `INTERNAL` fica reservado para falha do próprio N3X.
- O disjuntor é por instância de plugin (ADR-004): abre com 5 panics em 60 s ou com mais de 100
  zumbis (ajustável em `runtime.panic_breaker`), fica aberto até o restart do sidecar e DEVE gerar
  métrica, log e evento para o controller.
- Uma negação gRPC DEVE sair como resposta trailers-only com `grpc-status` diferente de 0 e
  `grpc-message`. Plugins NÃO DEVEM gerar resposta local crua, e nenhum erro gerado pelo Envoy sai
  como `OK`. A release é bloqueada se o teste de negação gRPC falhar em qualquer alvo.

**Justificativa:** o sidecar é a única porta de entrada da aplicação. Um plugin com defeito ou um
controller indisponível não podem derrubar o tráfego nem disfarçar uma negação como sucesso.

### II. Nenhum segredo sai do processo

- Valores decifrados NUNCA aparecem no bootstrap, no `/config_dump`, em log, em mensagem de erro, em
  métrica ou rótulo, nem na config enviada ao controller. Mensagens citam só o caminho do campo, a
  regra e o limite, nunca o valor.
- Segredos persistidos ficam só no formato `ENC[v1:...]`, com AES-256-GCM e dado associado
  `n3x:v1:<service>`. A chave fica sempre em arquivo montado, NUNCA em variável de ambiente. Os
  buffers de segredo são zerados depois do uso, por melhor esforço.
- A config do filtro do Envoy carrega só `config_path`: o `.so` lê, valida e decifra o próprio
  `n3x.yaml` (ADR-007).
- Logs NUNCA registram headers, corpos, query strings, mensagens de negação ou tokens de sessão.
- Todo pacote que lida com segredo, log, erro ou config enviada DEVE ter teste de canário
  (`n3x-canary-` com `secrettest.Leaks`) com zero ocorrências.

**Justificativa:** o sidecar convive no mesmo namespace de rede com a aplicação e com os coletores
de log do orquestrador. Segredo que sai do processo vira vazamento permanente.

### III. Superfície mínima e processo endurecido

- O admin do Envoy fica só no socket Unix `/tmp/n3x/admin.sock` (modo 0600), sem porta TCP. A
  `17090` expõe só `GET /metrics`, a `17001` fica reservada e sem uso, o `POST /drain` da `17021` só
  aceita loopback e o canal de controle confere o processo par por `SO_PEERCRED`.
- A aplicação NUNCA fica exposta: fora do Istio, ela escuta só em loopback e só o sidecar recebe
  tráfego.
- O processo roda sem root (UID e GID `17000`; `1337` no modo Istio). O `n3x run` como UID 0 DEVE
  recusar a subida (saída 8). A raiz é somente leitura, o `/tmp` é gravável e NUNCA é compartilhado
  com a aplicação, e o container roda sem capabilities.
- O `.so` só é carregado de `/usr/lib/n3x/libn3x.so`, dentro da imagem, NUNCA de fonte remota, com
  alvo e integridade conferidos contra o manifesto embutido.
- O path é normalizado antes da avaliação dos recortes, e os limites de headers (60 KiB e 100 itens)
  valem para toda requisição.

**Justificativa:** um processo comprometido na aplicação divide a rede com o sidecar. Cada porta,
endpoint ou permissão a mais vira caminho de ataque.

### IV. Configuração estática, fonte única e veredito único

- O `n3x.yaml` v1 é a única fonte de configuração do sidecar e dos plugins. Nenhum comportamento vem
  de variável de ambiente do operador. O entrypoint só lê, para detectar o ambiente, as variáveis
  que o orquestrador injeta, e só define internamente as variáveis do runtime Go e os marcadores dos
  processos do próprio container.
- A config é lida só na subida e só muda por redeploy. O hash reportado ao controller DEVE
  corresponder à config em uso.
- Uma única validação (`config.Load`, em `core/config`) roda no `n3x validate`, no `n3x run` e no
  `.so`, com o mesmo veredito para a mesma entrada. Divergência entre os três é defeito bloqueante.
- A decodificação é estrita: campo desconhecido é erro, `version` precisa ser `1` e campos novos só
  entram por adição.
- Sem fallback silencioso: valor inválido NUNCA vira padrão e a recusa é sempre explícita, com
  código de saída e código `N3X-Sxxx`, em vez de bypass. Sem limite de memória (nem cgroup nem
  `resources.memory`), ou com menos de 128 MiB, o sidecar NÃO DEVE subir.

**Justificativa:** previsibilidade. A config versiona junto com o serviço, um arquivo que passa na
CI sobe em produção, e nenhum valor muda sem deixar rastro.

### V. Prazo obrigatório e recursos limitados

- Nenhuma operação fica sem prazo (ADR-006). Os padrões são os do HLD: trabalho assíncrono 200 ms,
  conexão com a aplicação 1 s, requisição 15 s (em gRPC, o `grpc-timeout` do cliente até 1 h),
  chamada ao controller 5 s, login 10 s, health check 1 s, `envoy --mode validate` 10 s, Envoy pronto
  30 s, drenagem 20 s, encerramento dos plugins 2 s e SIGKILL 5 s depois do SIGTERM.
- O SDK só entrega `ctx` com prazo aos plugins. Só os streams com o controller não têm teto total.
- Todo recurso é limitado: até `max_inflight` trabalhos por instância (zumbis contam até terminar),
  buffer do controller com tamanho fixo e descarte do mais antigo, cotas por plugin em
  `runtime.plugin_limits`. Nada cresce sem limite.
- O runtime Go é calculado a partir do cgroup (ADR-008): `GOMAXPROCS = max(CPU, 2)`, sempre maior ou
  igual aos workers do Envoy, e `GOMEMLIMIT` em 45% da memória, com o overload manager agindo em 90%
  e 95%. As fórmulas são constantes do código e não são ajustáveis pelo operador.

**Justificativa:** Go não mata goroutines por fora. Sem prazo e sem limite, um upstream lento
acumula goroutines, memória e conexões até derrubar o sidecar.

### VI. Desempenho é contrato medido

- Metas do modo padrão: latência adicionada p50 ≤ 0,5 ms e p99 ≤ 2 ms a 1.000 rps, com carga
  open-loop abaixo do joelho da curva e comparação com o Envoy puro; sidecar ocioso com até 100 MiB e
  0,05 vCPU no container inteiro; pronto em até 5 s depois que o container sobe.
- O caminho comum da requisição (adaptador, executor, recortes, negação e telemetria) DEVE ter zero
  alocações, medidas com `-benchmem` e `testing.AllocsPerRun`.
- Cada requisição DEVE ter no máximo 5 entradas do Envoy no Go com trabalho da cadeia,
  independentemente do número de plugins. O body só cruza para o Go se alguma instância do recorte
  pedir, e uma instância fora do recorte não é chamada.
- O orçamento por parcela (ADR-009) só muda com base no benchmark. Regressão acima do orçamento
  bloqueia a release.
- O modo Istio tem metas próprias, definidas na SID-11.

**Justificativa:** o N3X só se justifica se custar menos do que reimplementar o comportamento em cada
serviço. Sem medição contínua, o custo cresce em silêncio.

### VII. Fronteiras de módulo e dependência verificadas

- O repositório entrega o binário `n3x` e o módulo `github.com/giustech/n3x-sidecar/core`, com
  `go.mod` próprio. Ele NUNCA entrega `.so` de produção, que sai sempre do `n3x-plugin build` do
  `n3x-builder`; o `.so` de desenvolvimento só existe com a build tag `n3x_dev`.
- Só `core/adapter/*`, com exatamente uma build tag de alvo (`n3x_envoy140` ou `n3x_istio131`),
  importa o SDK Go do Envoy. Executor, telemetria e config são neutros de alvo.
- O binário `n3x` é estático (`CGO_ENABLED=0`), sem grpc-go, sem o SDK do Envoy e sem
  `go-control-plane`. A CI confere isso com `go list -deps`.
- As decisões de cadeia (fases, grupos, recortes e efeitos) vêm do motor `chain` do `n3x-sdk`. O
  sidecar NÃO DEVE reimplementar regra que pertence a outro repositório. O `core/` e os plugins usam
  a mesma versão do `n3x-sdk`.
- Sem `init()` com efeito colateral, sem `os.Exit` ou `log.Fatal` em biblioteca, sem `os/exec`
  depois de iniciar o Envoy e sem estado global além do documentado no contrato. Toda goroutine
  criada pelo core tem `recover` e rótulos de pprof.

**Justificativa:** o `core/` é uma API consumida pelo builder dentro do processo do Envoy, onde um
erro fatal do Go derruba o proxy. Dependência fora do lugar quebra alvos, infla o binário ou fura o
isolamento.

### VIII. Test-First (NÃO NEGOCIÁVEL)

- O TDD é obrigatório em todo código de produção de `cmd/`, `internal/` e `core/`: o teste é escrito,
  aprovado e visto falhando antes do código (Red-Green-Refactor).
- São obrigatórios: `go test -race`; teste de tabela para cada regra de validação, código de saída e
  `reason`; fuzzing de todo parser; arquivos golden (bootstrap, logs por evento e `main` gerado),
  regenerados só com revisão; integração com o Envoy real na imagem exata de cada alvo e arquitetura,
  em HTTP/1.1, HTTP/2 e gRPC, incluindo negação gRPC e panic injetado; e o canário de segredo.
- A cobertura de linhas é portão da CI, por pacote: 90% na maioria; 85% em `internal/supervisor`,
  `internal/preflight`, `internal/health` e `core/grpcconn`; 80% nos adaptadores testados com ponte
  falsa.
- Todo defeito corrigido ganha o teste que o reproduz, como regressão.

**Justificativa:** a iteração de filtros, a contenção de falhas e o desligamento rodam dentro do
Envoy, onde a depuração é cara. Só testes escritos antes garantem que cada contrato é verificável.

### IX. Compatibilidade por adição

- A versão da imagem de runtime é a versão do `core/` e o `buildinfo.Version`, com as tags `vX.Y.Z`
  e `core/vX.Y.Z` no mesmo commit. Tags de imagem são imutáveis e as imagens base são fixadas por
  digest.
- Na subida, o sidecar aceita um `.so` cujo `core/` tenha a mesma major e minor igual ou menor que a
  da imagem; na série 0, a minor precisa ser igual. A comparação usa só a regra única do pacote
  `artifact`, nunca uma comparação própria.
- Contratos públicos ficam estáveis na major: subcomandos e flags, códigos de saída, códigos
  `N3X-Sxxx`, valores de `reason`, nomes e rótulos de métricas, eventos de log, campos do `n3x.yaml`,
  portas, caminhos, UID, `ENTRYPOINT` e `CMD`. Mudanças só entram por adição numa minor; remoção ou
  mudança de significado só numa major.

**Justificativa:** o entrypoint, o `.so` do operador, o builder e o controller evoluem em ritmos
diferentes. Só a evolução por adição permite atualizar um sem quebrar os outros.

## Restrições técnicas e de plataforma

- **Stack:** Go 1.26 ou superior; Envoy 1.40 (`envoyproxy/envoy`) e istio-proxy 1.31
  (`istio/proxyv2`), fixados por digest; SDK Go de dynamic modules do Envoy em pseudo-versão fixada
  pelo catálogo do `n3x-builder`; grpc-go só no `.so`, em `core/grpcconn`; YAML com
  `go.yaml.in/yaml/v3`, via `yamlconv` do `n3x-sdk`; `log/slog` com saída JSON; stats nativas do
  Envoy para as métricas `n3x_`.
- **Alvos:** `envoy-1.40` e `istio-1.31`, em amd64 e arm64, com um `.so` por commit do Envoy e por
  arquitetura.
- **Orquestradores:** Kubernetes 1.29 ou superior com sidecar nativo, ECS (EC2 e Fargate, rede
  `awsvpc`) e docker compose (`network_mode: service:`). A mesma imagem e o mesmo `n3x.yaml` DEVEM
  funcionar nos três, sem recurso exclusivo de nenhum.
- **Istio:** só compatibilidade, condicionada ao portão G0 do spike da SID-11. No modo Istio valem as
  regras próprias da SID-11: o Istio é dono do bootstrap, das portas, do health (`15021`), das
  métricas, do admin e da drenagem; o N3X não abre `17000`, `17021` nem `17090`; a memória
  recomendada é 512 MiB; e o modo fica fora do benchmark do modo padrão. Se o G0 falhar, o alvo é
  adiado sem bloquear a v1.
- **Portas e caminhos fixos:** `17000` (entrada), `17021` (health), `17090` (métricas), `17001`
  reservada; `/etc/n3x/n3x.yaml`, `/run/n3x/aes.key`, `/usr/lib/n3x/libn3x.so` e `/tmp/n3x`.
- **Fronteiras com outros repositórios:** contratos de plugin, motor `chain` e pacote `secret` são do
  `n3x-sdk`; login, sessão, registro, heartbeat e config efetiva são do `n3x-plugin-core`; geração do
  `.so`, análise estática e catálogo de alvos são do `n3x-builder`; o servidor gRPC e o painel são do
  `n3x-controller`.
- **Observabilidade:** logs JSON em stderr, uma linha por evento, com campos fixos; métricas `n3x_`
  sem rótulo com path, header ou valor do usuário; sem tracing distribuído na v1.
- **Fora da v1:** tráfego de saída, interceptação por iptables, injeção automática, tracing,
  observabilidade do tráfego da aplicação, autenticação do tráfego e recarga de config sem restart.

## Fluxo de desenvolvimento e portões de qualidade

- **Specs:** uma por FDD, em `specs/NN_nome` (rodar com `SPECIFY_FEATURE_DIRECTORY`), na ordem de 01 a
  12; cada uma depende só de specs de número menor, e a 12 é condicional ao portão G0. As fases
  `specify`, `clarify`, `plan`, `tasks` e `implement` rodam cada uma numa sessão própria.
- **Hipóteses:** itens marcados "(hipótese)" nas FDDs são validados no spike da fase 0, dentro do
  `/speckit-plan` da spec. Item reprovado obriga a rever a FDD antes de fechar o plano.
- **Git:** trunk-based, com branches curtas a partir de `main` e merge só por PR com a CI verde.
  Commits seguem Conventional Commits. As tags de release `vX.Y.Z` e `core/vX.Y.Z` saem no mesmo
  commit, com a árvore limpa.
- **CI (GitHub Actions):**
  - Em todo PR: unidade, `-race`, golden, lint, cobertura, `go list -deps`, fumaça amd64 e, quando o
    PR mexe em `core/adapter`, o cenário de negação gRPC do Istio (IST-03).
  - À noite e na pré-release: arm64, cenários Istio, benchmark open-loop na máquina dedicada
    `n3x-perf` (wrk2, Nighthawk ou ghz com HdrHistogram), testes de longa duração com o perfil de
    goroutines e fuzzing longo.
  - Nenhuma release sai sem todos esses portões verdes.
- **Lint:** `gofmt`, `go vet`, `staticcheck` e `gosec` em todo PR, no repositório inteiro; achado de
  severidade alta do `gosec` só passa com `#nosec` e motivo. O `govulncheck` gera relatório no PR e
  bloqueia a release.
- **Idioma:** specs, planos, tarefas, FDDs, ADRs e demais documentos em português do Brasil; o
  `README.md` em inglês; identificadores, campos, comandos, nomes de arquivo e branches em inglês.
- **Registro de decisões:** decisões aprovadas entram no texto da seção afetada e num bloco
  "Clarificações (AAAA-MM-DD)" do documento; ADRs ganham "Nota de revisão (AAAA-MM-DD)" no próprio
  arquivo. Só os arquivos `.md` valem como fonte.

## Governança

- Esta constituição prevalece sobre as demais práticas do repositório. Spec, plano ou tarefa que
  conflita com uma regra DEVE/NUNCA é corrigido; o princípio só muda por emenda.
- **Precedência dos documentos técnicos:** PRD, HLD e ADRs prevalecem sobre as FDDs, e a FDD que
  diverge é revista antes da spec dela. Exceções decididas em 2026-10-24, a favor das FDDs: a versão
  da imagem igual à do `core/` (SID-05, SID-08 e SID-11), as regras próprias do modo Istio (SID-11) e
  a cadência de CI.
- **Emendas:** feitas pelo `/speckit-constitution`, num PR próprio com commit
  `docs(constitution): ...`, com o Sync Impact Report atualizado e a propagação para os templates de
  `.specify/templates/`. Emenda que muda uma decisão de arquitetura também ganha nota de revisão no
  ADR correspondente.
- **Versionamento desta constituição:** MAJOR quando um princípio é removido ou redefinido de forma
  incompatível; MINOR quando um princípio ou seção é acrescentado ou ampliado; PATCH para
  esclarecimento e redação.
- **Conformidade:** o "Constitution Check" do `/speckit-plan` é portão antes da pesquisa da fase 0 e
  de novo depois do design da fase 1. O `/speckit-analyze` trata violação desta constituição como
  CRITICAL. Toda revisão de PR confere a conformidade, e toda exceção é justificada na seção
  "Complexity Tracking" do plano.
- **Referências de trabalho:** `README.md` (operação e implantação), `DRAFT_CONSTITUITION.md`
  (contexto de negócio que originou esta constituição), `docs/` (PRD, HLD, ADRs e FDDs) e
  `specs/README.md` (índice e ordem das specs).

**Versão**: 1.0.0 | **Ratificada em**: 2026-10-24 | **Última alteração**: 2026-10-24
