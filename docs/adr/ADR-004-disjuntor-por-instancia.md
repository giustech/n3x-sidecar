# ADR-004: Disjuntor por instância de plugin, por panics e por zumbis

Status: Aceito
Data: 2026-10-19

## Contexto
O FR-004 pede que panics repetidos desliguem o plugin até o próximo restart, sem definir o que é "repetido". Sem esse desligamento, um panic disparável por input vira negação de serviço: cada panic custa recover, captura de stack e log, e pode deixar o estado do plugin corrompido. Tirar o plugin do `n3x.yaml` exige redeploy, o que leva minutos ou horas. Um plugin que ignora o `ctx` causa o mesmo problema com goroutines que não terminam.

## Decisão
- O disjuntor de emergência é separado por instância de plugin (`instance`), não por plugin
- Abre com 5 panics numa janela deslizante de 60 s ou com mais de 100 zumbis (goroutines que passaram do prazo e não voltaram)
- Aberto: a instância não é chamada e segue a política dela (fail-closed nega com 503 ou `UNAVAILABLE`; fail-open pula)
- Fecha só no restart do sidecar
- Gera métrica (`n3x_plugin_breaker_open`), log e evento enviado ao controller pelo plugin core
- Limites configuráveis em `runtime.panic_breaker`

## Justificativa
- A resposta ao cliente é a mesma com ou sem o disjuntor; ele só corta o custo do panic e o risco de estado corrompido
- Age em milissegundos, enquanto o redeploy leva minutos ou horas

## Trade-off
- Um plugin pode ficar fora até o restart por um surto curto de panics
- Mais estado e mais lógica no caminho quente

## Requisitos relacionados
FR-004, FR-016; confiabilidade nos requisitos não funcionais.
