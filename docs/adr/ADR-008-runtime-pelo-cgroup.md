# ADR-008: Runtime calculado pelo cgroup e recusa sem limite de memória

Status: Aceito
Data: 2026-10-19

## Contexto
O runtime Go e o Envoy dividem CPU e memória no mesmo cgroup. `GOMAXPROCS` precisa ser maior ou igual ao número de workers do Envoy, e é preciso evitar OOM.

## Decisão
- `CPU` = `resources.cpus` ou o limite do cgroup, arredondado para cima, mínimo 1; `MEM` = `resources.memory` ou o limite do cgroup
- `envoy --concurrency` = `CPU`; `GOMAXPROCS` = `max(CPU, 2)`; `GOMEMLIMIT` = 45% de `MEM`; `GODEBUG=cgocheck=0`
- Overload manager com o monitor `cgroup_memory`: reduz o heap em 90% e para de aceitar conexões em 95%
- Sem limite de memória (nem cgroup nem `resources.memory`), o sidecar recusa subir com mensagem clara
- No modo Istio, o sidecar recusa subir se o `GOMAXPROCS` calculado ficar abaixo do `concurrency` do proxy

## Justificativa
- Sem limite de memória não há como proteger contra OOM
- Valores coerentes em Kubernetes, ECS e docker compose, sem variável de ambiente do operador

## Trade-off
- Containers sem limite de memória precisam de ajuste para usar o N3X
- Os percentuais são hipótese e serão ajustados pelo benchmark

## Requisitos relacionados
FR-001; desempenho e confiabilidade nos requisitos não funcionais.
