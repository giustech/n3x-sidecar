# ADR-002: Health `17021` no entrypoint e métricas `17090` no Envoy

Status: Aceito
Data: 2026-10-19

## Contexto
O ADR-006 do `n3x-sdk` fixou as portas de health (`17021`) e de métricas (`17090`), mas não quem as atende. O health precisa combinar o estado do Envoy, o health check da aplicação e a drenagem.

## Decisão
- `17021` é servido pelo entrypoint, com `/live` e `/ready`
- `/ready` responde 200 só quando o Envoy está pronto (`/ready` no admin), a aplicação passa no health check do `n3x.yaml` (`http`, `grpc` ou `tcp`) e o sidecar não está drenando
- No SIGTERM, `/ready` passa a responder 503 na hora, antes de qualquer outra etapa da drenagem
- `17090` é um listener do Envoy com rota exata `GET /metrics` reescrita para `/stats/prometheus` no admin; nenhum outro endpoint do admin é exposto
- No modo Istio, valem o health e o merge de métricas do Istio

## Justificativa
- O entrypoint já conduz a drenagem e controla o estado de pronto sem depender do Envoy
- Health check da aplicação sem as limitações do filtro de health do Envoy
- As métricas do `.so` já são stats do Envoy, então o próprio Envoy as expõe

## Trade-off
- Mais um servidor HTTP no entrypoint
- O health depende do entrypoint estar vivo, o que já é verdade porque ele é o PID 1

## Requisitos relacionados
FR-003; observabilidade nos requisitos não funcionais.
