# ADR-009: Orçamento de latência e recursos do sidecar

Status: Aceito
Data: 2026-10-19

## Contexto
O ADR-010 do `n3x-sdk` manda dividir as metas do sistema por componente no HLD de cada repositório. A meta do PRD é p50 de até 0,5 ms e p99 de até 2 ms de latência adicionada a 1.000 rps, com o sidecar ocioso em até 100 MB e 0,05 vCPU.

## Decisão
- Latência (p50 / p99): Envoy puro 0,30 / 1,00 ms; core do `.so` 0,15 / 0,50 ms; plugins síncronos (core + exemplo) 0,05 / 0,10 ms; folga de 0,40 ms no p99
- Ocioso: Envoy 45 MB e 0,02 vCPU; runtime Go + core 35 MB e 0,02 vCPU; plugins 20 MB e 0,01 vCPU
- 0 alocações por requisição no executor, nos recortes e na negação, e até 5 cruzamentos Envoy→Go por requisição
- Medição com carga open-loop abaixo do joelho da curva, contra o Envoy puro na mesma máquina
- Dimensionamento recomendado: 1 vCPU e 256 MiB para até 1.000 rps (hipótese)

## Justificativa
- Separa o custo do N3X do custo do próprio Envoy
- Dá um portão objetivo para a CI noturna

## Trade-off
- Números estimados, sem benchmark público de referência; serão revistos após a primeira medição

## Requisitos relacionados
Desempenho nos requisitos não funcionais; FR-003, FR-004.

## Nota de revisão (2026-10-22)
- A meta de cruzamentos passa a ser "até 5 entradas do Envoy no Go com trabalho da cadeia por requisição". Criação, destruição, pedaços de body e trailers sempre entram no Go pelo ABI de dynamic modules e são passagens O(1), sem alocação (SID-02, I-01; PRD atualizado)
- Unidade de memória: MiB (PRD atualizado)
- O orçamento ocioso ganha a parcela do entrypoint (supervisor e health): até 20 MiB e 0,005 vCPU (hipótese; SID-07 e SID-09). As parcelas somam 120 MiB e 0,055 vCPU, acima da meta: a meta dura continua sendo o total do container (100 MiB e 0,05 vCPU), e as parcelas são rebalanceadas pelo benchmark (regra S5 da SID-12) antes da v1
