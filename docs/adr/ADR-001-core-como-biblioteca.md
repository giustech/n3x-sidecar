# ADR-001: Core do `.so` como biblioteca, com o `.so` sempre gerado pelo `n3x-plugin build`

Status: Aceito
Data: 2026-10-19

## Contexto
O `libn3x.so` junta o core do sidecar com o plugin core e os plugins escolhidos pelo operador, e essa lista só é conhecida no Dockerfile dele. Era preciso decidir se o `n3x-sidecar` compila um `.so` próprio ou só fornece o código que o builder compila.

## Decisão
- O repositório entrega duas coisas:
  - o binário `n3x` (`cmd/n3x`), embutido na imagem `n3x/sidecar:<versão>-<alvo>`;
  - o módulo Go `core/` (executor, contenção de falhas, conexão compartilhada, telemetria e adaptadores por alvo).
- O `n3x-plugin build` gera um `main` que importa `core/`, o plugin core e os plugins e compila um único `libn3x.so`
- A imagem de runtime não traz `.so` padrão; o `.so` entra no estágio final do Dockerfile do operador
- Os pacotes `core/config` e `core/identity`, e o pacote `secret` do `n3x-sdk` (ADR-014 do `n3x-sdk`, que substituiu `core/secret`), são usados pelo entrypoint e pelo `.so`
- Os adaptadores por alvo ficam separados por build tag (`n3x_envoy140` e `n3x_istio131`)

## Justificativa
- Uma única forma de gerar o `.so`, sempre com verificação de símbolos e análise estática
- Mesma validação de config no entrypoint e no `.so`, sem lógica duplicada

## Trade-off
- Não dá para rodar a imagem `n3x/sidecar` sem passar pelo builder, nem para um teste rápido
- O módulo `core/` passa a ser uma API consumida pelo builder e precisa de versionamento cuidadoso

## Requisitos relacionados
FR-001, FR-004, FR-015.
