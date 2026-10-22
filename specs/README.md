# Specs do n3x-sidecar

Índice das especificações que serão desenvolvidas com o GitHub Spec Kit. Cada diretório corresponde a uma FDD de `docs/fdd/`, e a numeração é a ordem de execução: um grupo depende só de grupos com número menor.

Planejamento aprovado em 2026-10-22, depois da validação de consistência entre o PRD (`docs/prd/PRD-N3X.md`), o HLD (`docs/HLD.md`), os ADRs (`docs/adr/`) e as FDDs.

## Índice

| Spec | FDD | Escopo | Pacotes e pastas | Depende de |
| --- | --- | --- | --- | --- |
| [`01_config_e_cli`](01_config_e_cli/) | [SID-01](../docs/fdd/FDD-SID-01-n3x-yaml-e-cli-de-operador.md) (FDD 15) | Schema do `n3x.yaml` v1 e CLI de operador (`validate`, `encrypt`, `keygen`) | `core/config`, `cmd/n3x` | nenhum |
| [`02_bootstrap_envoy`](02_bootstrap_envoy/) | [SID-06](../docs/fdd/FDD-SID-06-gerador-de-bootstrap.md) (FDD 26) | Gerador do bootstrap do Envoy, config do filtro e admin em socket Unix | `internal/bootstrap`, `core/envoycfg` | 01 |
| [`03_adaptador_envoy_telemetria`](03_adaptador_envoy_telemetria/) | [SID-02](../docs/fdd/FDD-SID-02-adaptador-envoy-1-40-e-telemetria.md) (FDD 16) | Adaptador do alvo `envoy-1.40` e telemetria do `.so` | `core/adapter`, `core/adapter/envoy140`, `core/adapter/active`, `core/telemetry` | 01, 02 |
| [`04_executor_cadeia`](04_executor_cadeia/) | [SID-03](../docs/fdd/FDD-SID-03-executor-da-cadeia.md) (FDD 17) | Executor da cadeia de plugins e entrada do `.so` | `core/chain`, `core/entry`, `core/chain/chaintest`, `cmd/libn3x-dev`, `dev/` | 01, 03 |
| [`05_contencao_falhas`](05_contencao_falhas/) | [SID-04](../docs/fdd/FDD-SID-04-contencao-de-falhas-e-disjuntor.md) (FDD 18) | Contenção de falhas, prazos e disjuntor por instância | `core/fault`, `core/internal/ctxmark` | 02, 03, 04 |
| [`06_conexao_controller`](06_conexao_controller/) | [SID-10](../docs/fdd/FDD-SID-10-conexao-compartilhada.md) (FDD 38) | Conexão gRPC compartilhada com o controller | `core/grpcconn` | 01, 03, 04, 05 |
| [`07_preflight_identidade`](07_preflight_identidade/) | [SID-05](../docs/fdd/FDD-SID-05-pre-voo-identidade-e-manifesto.md) (FDD 25) | Pré-voo da subida: identidade da instância e conferência do `.so` | `internal/preflight`, `core/identity`, `core/buildinfo` | 01, 02, 03, 04 |
| [`08_supervisor_runtime_go`](08_supervisor_runtime_go/) | [SID-07](../docs/fdd/FDD-SID-07-supervisor-e-runtime-go.md) (FDD 27) | Supervisor do Envoy e runtime Go calculado pelo cgroup | `internal/supervisor`, `core/goruntime` | 01, 02, 07 |
| [`09_health_drenagem`](09_health_drenagem/) | [SID-09](../docs/fdd/FDD-SID-09-health-e-drenagem.md) (FDD 29) | Health `17021`, drenagem no SIGTERM, canal de controle e `n3x probe` | `internal/health`, `core/control`, `core/drain` | 02, 03, 04, 05, 06, 08 |
| [`10_imagens_runtime`](10_imagens_runtime/) | [SID-08](../docs/fdd/FDD-SID-08-imagens-de-runtime.md) (FDD 28) | Imagens de runtime do sidecar e conferência do ambiente | `images/runtime`, `images/tools/imageargs`, `internal/imagecheck`, `images/test` | 01, 07, 08 (09 para o `/ready`) |
| [`11_benchmark_orcamento`](11_benchmark_orcamento/) | [SID-12](../docs/fdd/FDD-SID-12-benchmark-e-orcamento.md) (FDD 50) | Benchmark open-loop e orçamento do sidecar | `bench/` | 01 a 10 |
| [`12_modo_istio`](12_modo_istio/) | [SID-11](../docs/fdd/FDD-SID-11-modo-istio-1-31.md) (FDD 49) | Modo Istio e alvo `istio-1.31` | `internal/istiomode`, `core/adapter/istio131`, `internal/istiofilter`, `deploy/istio/` | 01 a 10 |

Cada spec cobre uma FDD inteira. Nenhuma FDD aparece em duas specs.

## Situação de cada spec

| Spec | Situação | O que resolver antes de fechar o plan |
| --- | --- | --- |
| 01 | Pronta | Leitor do manifesto da FDD 22 (`n3x-builder`) entra como stub até existir |
| 02 | Pronta | Itens "conferir no spike" do bootstrap; validação com o `.so` real depois das specs 03 e 04 |
| 03 | Pronta | Spike do ABI do Envoy (S-01 a S-12) |
| 04 | Pronta | Refinos R1 a R10 no pacote `chain` do `n3x-sdk` (FDD 07) e capacidades da FDD 09 |
| 05 | Pronta | Nenhum item de spike |
| 06 | Pronta | Política de keepalive do controller (FDD 33, `n3x-controller`) |
| 07 | Pronta | Spike de identidade (ID-01 a ID-05); pacote `artifact` da FDD 22 |
| 08 | Pronta | Spike de cgroup e runtime Go (RT-01 a RT-05) |
| 09 | Pronta | Spike da drenagem (D-01 a D-08) |
| 10 | Pronta | Spike da imagem (IMG-01 a IMG-06); catálogo de alvos da FDD 19 |
| 11 | Pronta para a spec | A execução depende da máquina dedicada `n3x-perf` e do harness da FDD 48 (`n3x-sdk`) |
| 12 | **Condicional** | Portão G0 do spike Istio (IS-01 a IS-16): se falhar, o alvo `istio-1.31` é adiado (contrato 14 da SID-11) |

Os itens de spike da fase 0 rodam na pesquisa do `/speckit-plan` de cada spec. Se um item reprovar, a seção correspondente da FDD é revista antes de fechar o plano.

## Como usar com o Spec Kit

1. Rode `/speckit-constitution` antes da primeira spec: `.specify/memory/constitution.md` ainda é o template, e as FDDs contam com TDD como regra da constituição.
2. Em cada spec, rode `/speckit-specify` com `SPECIFY_FEATURE_DIRECTORY=specs/NN_nome` e a FDD como descrição. Sem isso, o Spec Kit cria um diretório novo no formato `NNN-nome`.
3. Leia o bloco **Clarificações (2026-10-22)** no fim da seção 1 da FDD. A lista "Refinos pedidos por FDDs posteriores" diz quais contratos de outras FDDs prevalecem sobre o texto dela. Por exemplo, a SID-03 recebe refinos da SID-04, da SID-05, da SID-09, da SID-10 e da SID-11.
4. Siga a ordem: `specify`, `clarify`, `plan`, `tasks` e `implement`, cada fase numa sessão própria.

## Referências

- PRD: [`docs/prd/PRD-N3X.md`](../docs/prd/PRD-N3X.md)
- HLD: [`docs/HLD.md`](../docs/HLD.md)
- ADRs: [`docs/adr/README.md`](../docs/adr/README.md)
- FDDs: [`docs/fdd/`](../docs/fdd/)
