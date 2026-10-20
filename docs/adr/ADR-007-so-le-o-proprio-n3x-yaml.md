# ADR-007: O `.so` lê o próprio `n3x.yaml`, sem segredos no bootstrap

Status: Aceito
Data: 2026-10-19

## Contexto
A config dos plugins precisa chegar do `n3x.yaml` ao `.so`. Embutir a config decifrada no bootstrap deixaria segredos em claro em disco e no `/config_dump` do admin. No modo Istio não há entrypoint gerando bootstrap, e o `EnvoyFilter` estático só consegue apontar para um caminho.

## Decisão
- A config do filtro contém só `config_path` e `key_path`
- O `.so` lê, valida e decifra o `n3x.yaml` sozinho, com o mesmo pacote `core/config` do entrypoint
- O entrypoint valida antes de subir e descarta os valores decifrados
- O mesmo caminho vale para o modo padrão e para o modo Istio

## Justificativa
- Nenhum segredo no bootstrap, no `/config_dump` ou em log
- Um único mecanismo para os dois modos

## Trade-off
- O `n3x.yaml` e a chave são lidos duas vezes na subida
- O `.so` precisa acessar os arquivos montados, inclusive no container do proxy do Istio

## Requisitos relacionados
FR-001, FR-004; segurança nos requisitos não funcionais.
