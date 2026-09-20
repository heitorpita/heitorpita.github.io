---
layout: page
title: Dados do experimento
permalink: /dados/
---

Tabelas completas do experimento de consumo energético descrito no relatório.
São **720 execuções**: 12 configurações (algoritmo × linguagem × versão) × 6 tamanhos de
entrada × 10 repetições.

Os dados brutos, os scripts de medição e o gerador destas tabelas estão em
[GoxRust_energy](https://github.com/heitorpita/GoxRust_energy). Esta página é gerada por `scripts/analise.py` a partir do csv
consolidado — não é editada à mão.

## Energia média por execução (J)

| Algoritmo | Linguagem | Versão | 10 | 100 | 1.000 | 10.000 | 50.000 | 100.000 |
|---|---|---|---|---|---|---|---|---|
| Flutuação | Go | clássica | 0,020 | 0,018 | 0,031 | 1,16 | 29,6 | 117,9 |
| Flutuação | Go | com extração | 0,019 | 0,019 | 0,032 | 1,16 | 30,2 | 120,5 |
| Flutuação | Rust | clássica | 0,019 | 0,019 | 0,030 | 1,29 | 31,2 | 123,0 |
| Flutuação | Rust | com limite | 0,015 | 0,017 | 0,023 | 0,863 | 21,7 | 87,4 |
| Inserção | Go | com trocas | 0,019 | 0,022 | 0,021 | 0,288 | 6,45 | 26,1 |
| Inserção | Go | com deslocamento | 0,018 | 0,018 | 0,020 | 0,128 | 2,70 | 11,7 |
| Inserção | Rust | com trocas | 0,018 | 0,018 | 0,020 | 0,136 | 2,89 | 11,6 |
| Inserção | Rust | com deslocamento | 0,018 | 0,017 | 0,019 | 0,089 | 1,81 | 7,43 |
| Seleção | Go | heapsort | 0,021 | 0,020 | 0,021 | 0,043 | 0,164 | 0,338 |
| Seleção | Go | clássica | 0,020 | 0,018 | 0,024 | 0,347 | 8,66 | 36,4 |
| Seleção | Rust | clássica | 0,018 | 0,016 | 0,020 | 0,311 | 7,15 | 28,8 |
| Seleção | Rust | heapsort | 0,018 | 0,018 | 0,019 | 0,024 | 0,085 | 0,160 |

## Desvio padrão da energia (J)

| Algoritmo | Linguagem | Versão | 10 | 100 | 1.000 | 10.000 | 50.000 | 100.000 |
|---|---|---|---|---|---|---|---|---|
| Flutuação | Go | clássica | 0,000 | 0,004 | 0,003 | 0,017 | 0,245 | 0,341 |
| Flutuação | Go | com extração | 0,003 | 0,006 | 0,004 | 0,017 | 0,223 | 0,432 |
| Flutuação | Rust | clássica | 0,009 | 0,006 | 0,005 | 0,061 | 1,41 | 3,05 |
| Flutuação | Rust | com limite | 0,005 | 0,005 | 0,005 | 0,011 | 0,220 | 0,405 |
| Inserção | Go | com trocas | 0,003 | 0,006 | 0,003 | 0,009 | 0,035 | 0,127 |
| Inserção | Go | com deslocamento | 0,004 | 0,006 | 0,000 | 0,004 | 0,023 | 0,190 |
| Inserção | Rust | com trocas | 0,004 | 0,004 | 0,000 | 0,013 | 0,027 | 0,078 |
| Inserção | Rust | com deslocamento | 0,004 | 0,005 | 0,003 | 0,003 | 0,024 | 0,051 |
| Seleção | Go | heapsort | 0,003 | 0,000 | 0,003 | 0,005 | 0,005 | 0,019 |
| Seleção | Go | clássica | 0,000 | 0,004 | 0,005 | 0,007 | 0,035 | 0,138 |
| Seleção | Rust | clássica | 0,004 | 0,005 | 0,000 | 0,007 | 0,070 | 0,238 |
| Seleção | Rust | heapsort | 0,004 | 0,004 | 0,003 | 0,005 | 0,005 | 0,000 |

## Duração média — `duration_time` (s)

| Algoritmo | Linguagem | Versão | 10 | 100 | 1.000 | 10.000 | 50.000 | 100.000 |
|---|---|---|---|---|---|---|---|---|
| Flutuação | Go | clássica | 0,0029 | 0,0030 | 0,0051 | 0,2006 | 4,9668 | 19,7987 |
| Flutuação | Go | com extração | 0,0029 | 0,0029 | 0,0050 | 0,1928 | 4,8785 | 19,4667 |
| Flutuação | Rust | clássica | 0,0030 | 0,0031 | 0,0051 | 0,2136 | 5,2446 | 20,8301 |
| Flutuação | Rust | com limite | 0,0027 | 0,0027 | 0,0042 | 0,1541 | 3,7950 | 15,1237 |
| Inserção | Go | com trocas | 0,0028 | 0,0031 | 0,0037 | 0,0518 | 1,1680 | 4,6401 |
| Inserção | Go | com deslocamento | 0,0029 | 0,0029 | 0,0033 | 0,0213 | 0,4368 | 1,8085 |
| Inserção | Rust | com trocas | 0,0027 | 0,0028 | 0,0031 | 0,0241 | 0,5005 | 1,9803 |
| Inserção | Rust | com deslocamento | 0,0028 | 0,0028 | 0,0031 | 0,0156 | 0,3067 | 1,2425 |
| Seleção | Go | heapsort | 0,0031 | 0,0029 | 0,0033 | 0,0070 | 0,0252 | 0,0496 |
| Seleção | Go | clássica | 0,0029 | 0,0028 | 0,0038 | 0,0583 | 1,4373 | 5,9206 |
| Seleção | Rust | clássica | 0,0028 | 0,0028 | 0,0036 | 0,0546 | 1,2558 | 5,0024 |
| Seleção | Rust | heapsort | 0,0028 | 0,0028 | 0,0029 | 0,0048 | 0,0138 | 0,0252 |

## Tempo de CPU em modo usuário — `user_time` (s)

| Algoritmo | Linguagem | Versão | 10 | 100 | 1.000 | 10.000 | 50.000 | 100.000 |
|---|---|---|---|---|---|---|---|---|
| Flutuação | Go | clássica | 0,0000 | 0,0000 | 0,0000 | 0,1960 | 4,9650 | 19,7990 |
| Flutuação | Go | com extração | 0,0000 | 0,0000 | 0,0000 | 0,1840 | 4,8710 | 19,4650 |
| Flutuação | Rust | clássica | 0,0009 | 0,0009 | 0,0027 | 0,2100 | 5,2368 | 20,8192 |
| Flutuação | Rust | com limite | 0,0010 | 0,0010 | 0,0014 | 0,1502 | 3,7873 | 15,1137 |
| Inserção | Go | com trocas | 0,0000 | 0,0000 | 0,0000 | 0,0450 | 1,1630 | 4,6360 |
| Inserção | Go | com deslocamento | 0,0000 | 0,0000 | 0,0000 | 0,0180 | 0,4260 | 1,8030 |
| Inserção | Rust | com trocas | 0,0007 | 0,0008 | 0,0011 | 0,0216 | 0,4971 | 1,9736 |
| Inserção | Rust | com deslocamento | 0,0009 | 0,0009 | 0,0011 | 0,0132 | 0,2990 | 1,2336 |
| Seleção | Go | heapsort | 0,0000 | 0,0000 | 0,0000 | 0,0000 | 0,0190 | 0,0430 |
| Seleção | Go | clássica | 0,0000 | 0,0000 | 0,0000 | 0,0500 | 1,4430 | 5,9250 |
| Seleção | Rust | clássica | 0,0009 | 0,0008 | 0,0014 | 0,0525 | 1,2498 | 4,9946 |
| Seleção | Rust | heapsort | 0,0008 | 0,0009 | 0,0010 | 0,0015 | 0,0108 | 0,0208 |

## Tempo de CPU em modo sistema — `system_time` (s)

| Algoritmo | Linguagem | Versão | 10 | 100 | 1.000 | 10.000 | 50.000 | 100.000 |
|---|---|---|---|---|---|---|---|---|
| Flutuação | Go | clássica | 0,0000 | 0,0000 | 0,0000 | 0,0000 | 0,0010 | 0,0250 |
| Flutuação | Go | com extração | 0,0000 | 0,0000 | 0,0000 | 0,0000 | 0,0060 | 0,0270 |
| Flutuação | Rust | clássica | 0,0001 | 0,0003 | 0,0003 | 0,0016 | 0,0052 | 0,0076 |
| Flutuação | Rust | com limite | 0,0000 | 0,0000 | 0,0009 | 0,0020 | 0,0056 | 0,0068 |
| Inserção | Go | com trocas | 0,0000 | 0,0000 | 0,0000 | 0,0000 | 0,0000 | 0,0060 |
| Inserção | Go | com deslocamento | 0,0000 | 0,0000 | 0,0000 | 0,0000 | 0,0010 | 0,0030 |
| Inserção | Rust | com trocas | 0,0003 | 0,0002 | 0,0002 | 0,0008 | 0,0016 | 0,0048 |
| Inserção | Rust | com deslocamento | 0,0001 | 0,0001 | 0,0000 | 0,0006 | 0,0056 | 0,0072 |
| Seleção | Go | heapsort | 0,0000 | 0,0000 | 0,0000 | 0,0000 | 0,0000 | 0,0010 |
| Seleção | Go | clássica | 0,0000 | 0,0000 | 0,0000 | 0,0000 | 0,0010 | 0,0130 |
| Seleção | Rust | clássica | 0,0001 | 0,0002 | 0,0006 | 0,0004 | 0,0040 | 0,0052 |
| Seleção | Rust | heapsort | 0,0002 | 0,0001 | 0,0000 | 0,0015 | 0,0012 | 0,0029 |

## Potência média do pacote (W)

| Algoritmo | Linguagem | Versão | 10 | 100 | 1.000 | 10.000 | 50.000 | 100.000 |
|---|---|---|---|---|---|---|---|---|
| Flutuação | Go | clássica | 7,04 | 5,90 | 6,07 | 5,77 | 5,95 | 5,96 |
| Flutuação | Go | com extração | 6,51 | 6,58 | 6,34 | 6,04 | 6,20 | 6,19 |
| Flutuação | Rust | clássica | 6,20 | 6,16 | 5,84 | 6,05 | 5,95 | 5,90 |
| Flutuação | Rust | com limite | 5,50 | 6,21 | 5,43 | 5,60 | 5,72 | 5,78 |
| Inserção | Go | com trocas | 6,90 | 6,98 | 5,77 | 5,56 | 5,53 | 5,63 |
| Inserção | Go | com deslocamento | 6,16 | 6,14 | 6,16 | 6,02 | 6,17 | 6,49 |
| Inserção | Rust | com trocas | 6,53 | 6,46 | 6,42 | 5,64 | 5,77 | 5,88 |
| Inserção | Rust | com deslocamento | 6,48 | 6,13 | 6,11 | 5,69 | 5,91 | 5,98 |
| Seleção | Go | heapsort | 6,85 | 6,92 | 6,33 | 6,11 | 6,50 | 6,82 |
| Seleção | Go | clássica | 7,04 | 6,33 | 6,27 | 5,95 | 6,03 | 6,14 |
| Seleção | Rust | clássica | 6,41 | 5,74 | 5,62 | 5,69 | 5,70 | 5,75 |
| Seleção | Rust | heapsort | 6,49 | 6,38 | 6,52 | 4,95 | 6,18 | 6,34 |


## Como reproduzir

```
scripts/criacao_lista.sh                 # gera as listas de entrada
scripts/implementacao.sh -r 10           # mede com o perf (10 execuções por configuração)
scripts/consolida.py                     # consolida os csv do perf em um só
scripts/analise.py                       # gera as figuras e estas tabelas
```
