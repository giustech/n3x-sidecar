# ADRs do n3x-sidecar

Decisões de arquitetura registradas no HLD do sidecar (`../HLD.md`). A numeração é própria deste repositório; ADRs de outros repositórios são citados com o nome dele (por exemplo, "ADR-006 do `n3x-sdk`"). Cada ADR traz contexto, decisão, justificativa, trade-off e requisitos relacionados do PRD (`../prd/PRD-N3X.md`).

| ADR | Título | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-core-como-biblioteca.md) | Core do `.so` como biblioteca, com o `.so` sempre gerado pelo `n3x-plugin build` | Aceito |
| [ADR-002](ADR-002-health-no-entrypoint-metricas-no-envoy.md) | Health `17021` no entrypoint e métricas `17090` no Envoy | Aceito, com nota de revisão (2026-10-22) |
| [ADR-003](ADR-003-subcomandos-do-entrypoint.md) | Subcomandos `run`, `validate`, `encrypt` e `keygen` no entrypoint | Aceito, com nota de revisão (2026-10-22) |
| [ADR-004](ADR-004-disjuntor-por-instancia.md) | Disjuntor por instância de plugin, por panics e por zumbis | Aceito, com nota de revisão (2026-10-22) |
| [ADR-005](ADR-005-grupo-paralelo-por-order.md) | Grupo paralelo por fase e `order` | Aceito |
| [ADR-006](ADR-006-prazo-obrigatorio.md) | Prazo obrigatório em toda operação | Aceito, com nota de revisão (2026-10-22) |
| [ADR-007](ADR-007-so-le-o-proprio-n3x-yaml.md) | O `.so` lê o próprio `n3x.yaml`, sem segredos no bootstrap | Aceito, com nota de revisão (2026-10-22) |
| [ADR-008](ADR-008-runtime-pelo-cgroup.md) | Runtime calculado pelo cgroup e recusa sem limite de memória | Aceito, com nota de revisão (2026-10-22) |
| [ADR-009](ADR-009-orcamento-do-sidecar.md) | Orçamento de latência e recursos do sidecar | Aceito, com nota de revisão (2026-10-22) |
