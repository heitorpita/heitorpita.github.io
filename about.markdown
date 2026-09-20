---
layout: page
title: Sobre
permalink: /about/
---

Sou estudante de computação e este blog é onde eu registro experimentos, medições e notas
de estudo — principalmente as coisas que só ficam claras depois de rodar.

## O experimento de energia

A postagem [Dois jeitos de ordenar, duas contas de luz]({% post_url 2026-09-20-energia-ordenacao-go-rust %})
é o relatório de um trabalho da disciplina de Tópicos Avançados em Computação, feito em
dupla com **Gabriel Moura**. A pergunta era quanto de energia se economiza ao escrever uma
segunda versão, mais cuidadosa, de um algoritmo de ordenação — e a resposta foi medida, não
estimada: 720 execuções instrumentadas com `perf` e os contadores RAPL do processador.

Todo o material está aberto:

- **Código, dados e scripts:** [github.com/heitorpita/GoxRust_energy](https://github.com/heitorpita/GoxRust_energy)
- **Tabelas completas:** [dados do experimento](/dados/)
- **Fonte deste site:** [github.com/heitorpita/heitorpita.github.io](https://github.com/heitorpita/heitorpita.github.io)
