# ADR-003: Subcomandos `run`, `validate`, `encrypt` e `keygen` no entrypoint

Status: Aceito
Data: 2026-10-19

## Contexto
O HLD de plataforma deixou pendente a ferramenta que gera os valores `ENC[...]`. Também é útil validar um `n3x.yaml` na CI do serviço, antes do deploy.

## Decisão
O binário `n3x` da imagem do sidecar tem quatro subcomandos:
- `n3x run [--config <arquivo>]`: padrão do `ENTRYPOINT`; sobe e supervisiona o Envoy
- `n3x validate --config <arquivo> --so <libn3x.so>`: valida o YAML e os schemas dos plugins do manifesto, sem subir o Envoy
- `n3x encrypt --key <arquivo>`: lê o segredo do stdin e imprime `ENC[v1:...]`
- `n3x keygen`: gera uma chave AES-256 aleatória em base64

Uso típico: `docker run --rm -i -v $PWD/aes.key:/k n3x/sidecar:<versão>-<alvo> encrypt --key /k`.

## Justificativa
- Uma ferramenta só, já presente na imagem, sem instalar nada
- Mesmo código de validação e de criptografia do runtime

## Trade-off
- Gerar `ENC[...]` exige ter a imagem do sidecar à mão
- O segredo em claro passa pelo stdin do operador

## Requisitos relacionados
FR-001; segurança nos requisitos não funcionais.
