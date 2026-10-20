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
