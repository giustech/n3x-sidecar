# ADR-006: Prazo obrigatório em toda operação

Status: Aceito
Data: 2026-10-19

## Contexto
Sem prazo, um upstream lento segura goroutines, memória e conexões até derrubar o sidecar. Go não consegue matar uma goroutine por fora.

## Decisão
- Nenhuma operação do sidecar fica sem prazo. Padrões: trabalho assíncrono 200 ms, conexão com a aplicação 1 s, requisição à aplicação 15 s, chamada ao controller 5 s, login 10 s, health check 1 s, validação do bootstrap 10 s, Envoy pronto 30 s, drenagem 20 s, encerramento dos plugins 2 s e SIGKILL 5 s após o SIGTERM
- O SDK só entrega `ctx` com prazo aos plugins
- A conexão compartilhada impõe `min(prazo do ctx, 5 s)` num interceptor do core
- O `n3x-plugin build` recusa `context.Background()`, `context.TODO()` e chamadas bloqueantes sem contexto em código de plugin (regra definida aqui, implementada no builder)
- Até 1.000 trabalhos simultâneos por instância; os zumbis contam até terminarem de fato

## Justificativa
- Toda requisição termina, e um plugin lento não acumula recursos sem limite

## Trade-off
- Prazos padrão podem ser curtos para alguns casos e precisam de ajuste no `n3x.yaml`
- A análise estática pode recusar código legítimo e exigir reescrita

## Requisitos relacionados
FR-002, FR-003, FR-004, FR-012; confiabilidade nos requisitos não funcionais.

## Nota de revisão (2026-10-22)
- Requisição gRPC à aplicação: o prazo é o `grpc-timeout` do cliente, com teto de 1 h; sem ele, vale `app.timeouts.request` (15 s); o estouro sai como `DEADLINE_EXCEEDED` (SID-06)
- Chamada unária ao controller: o teto é `min(prazo do ctx, controller.timeouts.call)`, com padrão de 5 s e faixa de 100 ms a 60 s; o login usa `controller.timeouts.login` (10 s); stream não tem teto total (SID-10, L-08)
- Ordem do desligamento: os 2 s de encerramento dos plugins começam antes do SIGTERM ao Envoy, pelo canal de controle, e o SIGTERM só sai com a confirmação do `.so` ou em 3 s; o SIGKILL continua 5 s depois do SIGTERM ao Envoy; total máximo de `runtime.drain_timeout` mais 9 s (SID-09)
