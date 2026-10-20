# ADR-005: Grupo paralelo por fase e `order`

Status: Aceito
Data: 2026-10-19

## Contexto
A ordem de execução dos plugins importa muito. Plugins independentes que fazem I/O na mesma fase pagariam a soma das latências se rodassem em sequência.

## Decisão
- Dentro de uma fase, os plugins rodam em ordem crescente de `order`
- Instâncias com a mesma fase e o mesmo `order` formam um grupo paralelo:
  - a parte síncrona roda em sequência no worker do Envoy, em ordem alfabética de `instance`;
  - os trabalhos `Go()` rodam juntos, e o grupo espera todos, cada um com o próprio prazo;
  - ninguém do grupo vê as mudanças do outro, e as mudanças são aplicadas no fim, em ordem alfabética de `instance` (vale a última);
  - negação vence: se alguém nega ou falha com fail-closed, vale a negação do primeiro em ordem alfabética e o `ctx` dos demais é cancelado.
- Na fase `route`, `order` repetido é recusado na validação
- Na resposta, os grupos rodam em ordem inversa, com as mesmas regras

## Justificativa
- O tempo do grupo é o do trabalho mais lento, não a soma
- Resultado determinístico, que não depende de quem termina primeiro
- Duas mudanças de rota em paralelo não teriam resultado correto

## Trade-off
- Plugins do mesmo grupo não conseguem usar o resultado um do outro
- Regras a mais para o autor de config entender

## Requisitos relacionados
FR-004, FR-012. Exige atualização do FR-004 no PRD.
