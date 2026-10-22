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

## Nota de revisão (2026-10-22)
- Sem limite de CPU no cgroup e sem `resources.cpus`: `CPU = min(afinidade, 2)`, com o aviso `cpu_unlimited`; o teto de 2 é revisto pelo benchmark (SID-07 e SID-12)
- `resources` e o cgroup valem juntos: vale o menor dos dois; memória abaixo de 128 MiB recusa a subida e abaixo de 256 MiB só avisa (SID-07)
- O exemplo de `GOMAXPROCS` do FR-001 no PRD foi alinhado a esta decisão
