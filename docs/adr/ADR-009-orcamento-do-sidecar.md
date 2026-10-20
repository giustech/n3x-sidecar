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
