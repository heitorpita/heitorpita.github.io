---
layout: post
title:  "Ordenar em joules: seis otimizações, duas máquinas, uma economia que sobreviveu"
date:   2026-09-20 09:00:00 -0300
categories: energia algoritmos experimento
---

**Gabriel Moura e Heitor Pita**  
Engenharia de Software Sustentável — Projeto 1, medindo software.  
Código, dados e scripts: [github.com/heitorpita/GoxRust_energy](https://github.com/heitorpita/GoxRust_energy)

<style>
.post-content figure { margin: 2em 0; }
.post-content figure img { width: 100%; height: auto; display: block; border: 1px solid #e1e0d9; border-radius: 4px; }
.post-content figcaption { font-size: 0.85em; color: #52514e; margin-top: 0.6em; line-height: 1.5; }
.post-content table { font-size: 0.82em; width: 100%; }
.post-content table td, .post-content table th { padding: 6px 10px; }
</style>

## A pergunta

Big-O conta operações. A conta de luz cobra joules. Entre a operação que você soma no papel
e o elétron que sai da tomada existe um compilador, um runtime, uma hierarquia de cache e um
governador de frequência, e nenhum deles aparece no somatório.

Queríamos responder uma coisa concreta: quando reescrevemos um algoritmo de ordenação com
uma ideia a mais, quanto essa ideia economiza de energia de verdade? Implementamos os três
métodos pedidos (flutuação, inserção e seleção) em duas versões cada, em Go e em Rust, e
medimos energia, tempo de parede e tempo de CPU com o `perf` lendo os contadores RAPL.

No meio do caminho tivemos acesso a uma segunda máquina, de outra geração, e resolvemos
rodar a bateria inteira de novo nela. Foi a melhor decisão do trabalho. São 1.020 execuções,
10 por célula, somando 16,5 kJ de energia medida, e a resposta à pergunta acima mudou de
sinal três vezes entre uma máquina e outra.

Para dar escala antes de entrar nos detalhes: no notebook mais novo, ordenar uma lista de
100 mil inteiros custou 189,1 J em 14,5 s com a flutuação ingênua em Go, e 0,215 J em 0,017 s
com o heapsort em Rust. Um fator de 880 entre a pior e a melhor configuração medida. A maior
diferença que encontramos entre linguagens, no mesmo algoritmo, foi 3,28.

## O cenário

Duas plataformas separadas por cinco anos de arquitetura Intel. A distância entre elas é o
que dá valor ao conjunto de dados, e é também o que limita o que dá para concluir.

| Item | Máquina A · Dell | Máquina B · HP |
|---|---|---|
| Modelo | Dell Inspiron 15 3511 | notebook consumer, plataforma Broadwell-U |
| CPU | Intel Core i5-1135G7 @ 2,40 GHz (Tiger Lake, 2020) | Intel Core i5-5200U @ 2,20 GHz (Broadwell, 2015) |
| Núcleos / threads | 4 / 8 | 2 / 4 |
| Cache L3 | 8 MiB | 3 MiB |
| Memória | 8 GiB DDR4-3200 | 11 GiB |
| Sistema | Linux (saída de `lshw`); distribuição e kernel não registrados | Debian 12 bookworm, kernel 6.1.0-52-amd64 |
| Governador / turbo | não registrado | `intel_cpufreq`, `schedutil`, 0,5–2,7 GHz, turbo ativo |
| Toolchain | não registrado | `rustc`/`cargo` 1.94.0; versão do Go não registrada |
| Domínios RAPL lidos | `power/energy-pkg/` | pkg, cores, dram |
| Potência média do pacote | ≈ 12,3 W | ≈ 6,0 W |
| Tamanhos medidos | 10.000 · 50.000 · 100.000 | 10 · 100 · 1.000 · 10.000 · 50.000 · 100.000 |
| Execuções | 360 (12 programas × 3 tamanhos × 10) | 660 (11 programas × 6 tamanhos × 10) |

As lacunas marcadas como não registrado são reais e atrapalham. O arquivo de specs do Dell é
uma saída de `lshw`, que descreve hardware e não diz nada sobre distribuição, kernel,
governador ou versão de compilador. O do HP é bem mais completo, e mesmo assim registra `go:
não instalado`, apesar das 330 execuções de binários Go medidas naquela máquina. Sempre que
este post disser "máquina", leia "plataforma inteira": hardware, sistema e compilador juntos.
Não temos como separar as três coisas com os dados que coletamos.

O outro número que importa da tabela é a potência média do pacote, 12,3 W contra 6,0 W. O
Dell sustenta o dobro do consumo instantâneo do HP, e isso volta no fim dos resultados.

## O que foi implementado

Doze programas de linha de comando, seis por linguagem. Cada um lê `n` na primeira linha e os
`n` inteiros na segunda, ordena em memória e imprime o resultado, de modo que o custo de
entrada e saída seja o mesmo para todos. Nenhum usa a ordenação da biblioteca padrão.

Antes de qualquer comparação, tivemos que arrumar o pareamento. Os arquivos se chamam `v1` e
`v2` nas duas linguagens, e o nome não corresponde ao que o código faz: `selection_v1.go`
monta um heap com `container/heap` e é heapsort Θ(n log n), enquanto `selection_sort_v1.rs` é
a seleção clássica do mínimo, Θ(n²). As versões `v2` invertem os papéis. Comparar os arquivos
homônimos devolveria uma razão de 73× "a favor de Go" que é pura diferença de complexidade
assintótica, reprodutível e sem significado nenhum. Reagrupamos por família semântica, pelo
que o código faz.

| Par comparado | Fonte Go | Fonte Rust | O que muda na segunda versão |
|---|---|---|---|
| Flutuação | `bubble_v1.go` → `bubble_v2.go` | `bubble_sort_v1.rs` → `bubble_sort_v2.rs` | Go: flag de troca e extração do maior por `slices.Insert`. Rust: limite do laço externo recua até a última troca. |
| Inserção | `insert_v1.go` → `insert_v2.go` | `insertion_sort_v1.rs` → `insertion_sort_v2.rs` | Troca a cada passo dá lugar a deslocamento com a chave em registrador. |
| Seleção | `selection_v2.go` → `selection_v1.go` | `selection_sort_v1.rs` → `selection_sort_v2.rs` | Varredura do mínimo dá lugar a heapsort, Θ(n²) para Θ(n log n). |

**Flutuação.** A versão ingênua faz `n` passadas completas, sem parada antecipada, e trabalha
igual mesmo sobre um vetor já ordenado. Em Rust, a segunda versão guarda o índice da última
troca: tudo à direita dele já está no lugar, então a passada seguinte encolhe até ali. Menos
comparações executadas, menos instruções buscadas e decodificadas, menos joules — era a
hipótese. Em Go a ideia foi a mesma, implementada de outro jeito: flag de troca mais
transferência do maior elemento para um vetor de saída com `slices.Insert(sorted, 0, ...)`.
Inserir na posição 0 desloca todo o conteúdo e custa O(n) por remoção, o que acrescenta um
trabalho quadrático de movimentação de memória que não existia antes. Essa dupla mede
linguagem e decisão de implementação ao mesmo tempo, e é assim que ela precisa ser lida.

**Inserção.** A versão por troca desce a chave posição a posição, trocando com o vizinho: três
atribuições em memória por posição percorrida (`aux = a[j]; a[j] = a[j-1]; a[j-1] = aux`), e a
condição do laço ainda relê `a[j]`, que acabou de ser escrito. A versão por deslocamento
guarda a chave numa variável local, empurra os maiores uma casa à direita com uma atribuição
por posição, e grava a chave uma vez só no fim. O número de comparações é idêntico nos dois
lados. O que cai para cerca de um terço é o número de escritas, e a hipótese era que menos
tráfego com o cache L1 e menos pressão no *store buffer* apareceriam no contador.

**Seleção.** A versão clássica procura o mínimo do sufixo a cada iteração: `n(n-1)/2`
comparações, sempre. Em Go ela foi escrita com `slices.Min`, `slices.Index` e `slices.Delete`,
o que percorre o vetor umas três vezes por elemento selecionado em vez de uma. O heapsort
constrói o heap em O(n) e extrai o extremo `n` vezes, cada extração custando O(log n). Para
n = 100.000, são cerca de 5 × 10⁹ comparações contra 3,4 × 10⁶.

## Como medimos

Cada execução é envolvida por:

```
perf stat -x';' -a -e power/energy-pkg/,duration_time \
    /usr/bin/time -f '%U;%S' ./binario < lista.in
```

Três decisões estão escondidas nessa linha. A primeira é o `-a`, que não é opcional: o
contador `power/energy-pkg/` vem da PMU `power`, que tem escopo de pacote e não de tarefa, e
pedir a energia só do processo devolve `<not supported>`. A consequência atravessa o post
inteiro. O que medimos é a energia do pacote durante a janela de execução, e nenhuma frase
aqui diz "o algoritmo X consome Y joules". A segunda é que o tempo de CPU vem do GNU `time`,
porque em modo `-a` os eventos `user_time` e `system_time` do `perf` somam a máquina inteira.
A terceira é que a compilação fica fora da janela: `go build` e `rustc -O -C debuginfo=0`
rodam antes de qualquer medição.

Dez execuções por célula, com média e desvio-padrão populacional. As razões entre linguagens
trazem intervalo de confiança de 95% por *bootstrap* com 20 mil reamostragens. Uma lista
aleatória por tamanho, gerada uma vez por `geracoes_listas/geraEntrada.cpp` e reutilizada em
todas as execuções das duas máquinas, o que elimina a variação de entrada e também significa
que todo resultado aqui vale para uma permutação, não para o caso médio.

Seguindo a lista de *freeze your settings* do material do curso, o que garantimos foi a mesma
entrada, os mesmos fontes, os mesmos scripts, a ordem fixa de execução e a compilação fora da
janela. Verificamos depois que não há deriva ao longo das dez repetições: a razão entre a
primeira execução e a média das nove seguintes é 1,000 nas duas máquinas. O que não foi
controlado nem registrado: brilho de tela, rede, bateria, temperatura ambiente, governador no
Dell e processos de segundo plano, sem pausa de resfriamento entre execuções. E nenhuma linha
de base ociosa foi medida em nenhuma das duas máquinas, que é a lacuna mais séria do conjunto.

## Resultados

### As magnitudes

<figure>
  <a href="{{ '/assets/img/energia/fig1-energia-100k.svg' | relative_url }}"><img src="{{ '/assets/img/energia/fig1-energia-100k.svg' | relative_url }}" alt="Barras horizontais da energia média por configuração em n = 100.000, um painel por máquina"></a>
  <figcaption>Figura 1 — Entre a melhor e a pior configuração há um fator de 880 no Dell e de 780 no HP; nenhuma diferença entre linguagens passa de 3,3. Energia média por execução em n = 100.000, escala logarítmica, barra de erro de um desvio-padrão sobre as 10 execuções.</figcaption>
</figure>

| Par | Ling. | Versão | Dell · E (J) | Dell · t (s) | Dell · P (W) | HP · E (J) | HP · t (s) | HP · P (W) | µJ/elem (HP) |
|---|---|---|---|---|---|---|---|---|---|
| Flutuação | Go | ingênua | 189,121 ± 3,162 | 14,525 ± 0,049 | 13,02 | 118,647 ± 5,062 | 19,969 ± 0,236 | 5,94 | 1.186 |
| Flutuação | Go | com extração | 171,855 ± 2,872 | 12,296 ± 0,055 | 13,98 | 124,724 ± 5,532 | 19,760 ± 0,263 | 6,31 | 1.247 |
| Flutuação | Rust | ingênua | 126,858 ± 1,568 | 10,356 ± 0,016 | 12,25 | 123,009 ± 2,890 | 20,830 ± 0,130 | 5,90 | 1.230 |
| Flutuação | Rust | com limite | 129,005 ± 1,291 | 10,462 ± 0,026 | 12,33 | 87,376 ± 0,385 | 15,124 ± 0,010 | 5,78 | 874 |
| Inserção | Go | por troca | 34,918 ± 0,423 | 3,097 ± 0,001 | 11,28 | 29,300 ± 0,476 | 4,790 ± 0,022 | 6,12 | 293 |
| Inserção | Go | por deslocamento | 9,569 ± 0,085 | 0,705 ± 0,001 | 13,57 | não executado | — | — | — |
| Inserção | Rust | por troca | 10,649 ± 0,038 | 0,866 ± 0,001 | 12,30 | 11,645 ± 0,074 | 1,980 ± 0,002 | 5,88 | 116 |
| Inserção | Rust | por deslocamento | 11,584 ± 0,114 | 0,981 ± 0,001 | 11,81 | 7,428 ± 0,049 | 1,243 ± 0,004 | 5,98 | 74,3 |
| Seleção | Go | clássica | 38,796 ± 0,450 | 3,241 ± 0,002 | 11,97 | 35,823 ± 0,265 | 5,928 ± 0,013 | 6,04 | 358 |
| Seleção | Go | heapsort | 0,445 ± 0,023 | 0,033 ± 0,002 | 13,46 | 0,375 ± 0,101 | 0,053 ± 0,006 | 7,00 | 3,75 |
| Seleção | Rust | clássica | 32,299 ± 0,392 | 2,408 ± 0,001 | 13,41 | 28,759 ± 0,226 | 5,002 ± 0,011 | 5,75 | 288 |
| Seleção | Rust | heapsort | 0,215 ± 0,007 | 0,017 ± 0,001 | 12,38 | 0,160 ± 0,000 | 0,025 ± 0,000 | 6,34 | 1,60 |

<!-- TODO: acrescentar colunas user/sys por célula (o enunciado, item 3, exige os três tempos).
     O GNU time já coleta; falta só o analise/ emitir. Confirmar também os nomes dos SVGs. -->

Os programas são monothread e limitados por CPU, sem espera de disco nem de rede, e o tempo
de CPU acompanha o tempo de parede em todas as células. As colunas `user` e `sys` completas,
para os seis tamanhos, estão na página de dados.

### O crescimento e o piso do instrumento

<figure>
  <a href="{{ '/assets/img/energia/fig2-escalabilidade.svg' | relative_url }}"><img src="{{ '/assets/img/energia/fig2-escalabilidade.svg' | relative_url }}" alt="Curvas log-log de energia em função do tamanho da entrada, um painel por par de versões"></a>
  <figcaption>Figura 2 — Entre n = 10.000 e n = 100.000 a energia das famílias quadráticas cresce com expoente medido de 1,89 a 2,00, contra o 2 da teoria; abaixo de n = 10.000 o que a curva mostra é o piso do instrumento, marcado em cinza. Eixos logarítmicos.</figcaption>
</figure>

Ver a teoria aparecer com essa limpeza na medição de energia foi uma boa surpresa. O trecho
plano à esquerda de cada painel, porém, não diz nada sobre algoritmos. Todas as células entre
n = 10 e n = 1.000 medem cerca de 0,020 J em 3,3 ms, que é o custo de subir o processo e ler
a entrada, e o coeficiente de variação nessa faixa chega a 47%. Nada abaixo de n = 10.000
deve ser interpretado.

O piso também contamina o heapsort nos tamanhos grandes. Os 0,160 J do heapsort em Rust no HP
incluem os mesmos 0,020 J de custo fixo, ou 12,8% do total, e o expoente medido de 0,71 a
0,77 fica bem abaixo do ~1,05 esperado de `n log n` por causa disso. As cifras de heapsort
valem como limite superior do custo do algoritmo. Isso tem uma consequência agradável para a
conclusão principal: a economia de 99% que reportamos mais adiante está subestimada, porque o
lado barato da comparação é o que carrega proporcionalmente mais sobrecarga.

### A reprodutibilidade

<figure>
  <a href="{{ '/assets/img/energia/fig3-boxplot-100k.svg' | relative_url }}"><img src="{{ '/assets/img/energia/fig3-boxplot-100k.svg' | relative_url }}" alt="Boxplots do desvio percentual de cada execução em relação à mediana da sua célula"></a>
  <figcaption>Figura 3 — A dispersão é baixa no Dell (quase todas as células abaixo de 1,5%) e maior nas células de Go no HP, que chegam a 4,4%. Desvio percentual de cada execução em relação à mediana da própria célula, em n = 100.000.</figcaption>
</figure>

Em joules absolutos as caixas somem, porque as células estão a três ordens de grandeza umas
das outras; normalizando pela mediana, todas ficam comparáveis. A maioria das células fica
abaixo de 1,5% de coeficiente de variação, o que para dez execuções sem controle de ambiente
está bom.

Duas exceções merecem registro, porque o post depende delas. As duas células de flutuação em
Go no HP têm CV de 4,3% e 4,4%, acima do limiar de 3% a partir do qual costumamos suspeitar
de interferência em vez de propriedade do programa. A diferença de 5% entre elas ainda
sobrevive quando se usa o erro-padrão da média em vez do desvio bruto, mas é o resultado mais
frágil que reportamos. E o heapsort em Go no HP tem 0,375 ± 0,101 J, ou seja, 27% de
dispersão sobre um valor que já está a poucos passos de contador do piso. Nenhuma leitura
fina sobre heapsort no HP se sustenta.

### Energia e tempo

<figure>
  <a href="{{ '/assets/img/energia/fig4-energia-x-duracao.svg' | relative_url }}"><img src="{{ '/assets/img/energia/fig4-energia-x-duracao.svg' | relative_url }}" alt="Dispersão de energia contra tempo de parede para todas as células, log-log, com duas retas de potência constante"></a>
  <figcaption>Figura 4 — Cada máquina ocupa uma faixa estreita em torno de uma potência praticamente constante, 12,3 W no Dell e 6,0 W no HP, e por isso o ranking de energia é quase idêntico ao de tempo dentro de cada máquina. Células com n ≥ 10.000, eixos logarítmicos.</figcaption>
</figure>

Os pontos caem sobre duas retas de inclinação 1, uma por máquina, que é o esperado quando a
potência do pacote varia pouco. Vale desconfiar um pouco do próprio resultado, no entanto. A
medição é de pacote, e o pacote consome mesmo quando o programa faz pouco: o piso de 0,020 J
mostra que existe uma parcela fixa razoável dentro de cada célula. Quanto maior essa parcela,
mais a relação `E = P̄ × t` vira aritmética em vez de achado experimental, porque uma potência
de base constante força o alinhamento sozinha. Sem a linha de base ociosa, não sabemos de que
tamanho ela é.

A comparação entre as duas faixas é o que o gráfico tem de mais interessante. O Dell executa
a flutuação ingênua em Go 1,37× mais rápido que o HP e gasta 1,59× mais energia para fazer o
mesmo trabalho. Mais rápido não é mais verde quando a plataforma muda: a potência do
hardware pode anular o ganho de tempo, e aqui anula com sobra.

Dentro de cada máquina a potência também não é tão constante quanto a Figura 4 sugere à
primeira vista. As células do Dell vão de 11,28 W a 13,98 W, uma faixa de 24%, e é
exatamente nos resíduos dessa faixa que moram os casos interessantes da próxima seção.

## O que economizou, e o que só economizou numa das máquinas

<figure>
  <a href="{{ '/assets/img/energia/fig5-ganho.svg' | relative_url }}"><img src="{{ '/assets/img/energia/fig5-ganho.svg' | relative_url }}" alt="Barras divergentes com a variação percentual de energia da segunda versão em relação à primeira, lado a lado para as duas máquinas"></a>
  <figcaption>Figura 5 — Os três pares de otimização de constante medidos nas duas máquinas trocaram de sinal entre elas; o par que muda a classe de complexidade economizou cerca de 99% nas quatro combinações. Variação percentual da energia da segunda versão em relação à primeira, n = 100.000; negativo é economia.</figcaption>
</figure>

Esta é a figura do trabalho, e a resposta ao item 1 do enunciado é mais estranha do que
esperávamos.

| Par | Dell | HP |
|---|---|---|
| Flutuação Go: ingênua → com extração | −9,1% | +5,1% |
| Flutuação Rust: ingênua → com limite | +1,7% | −29,0% |
| Inserção Rust: por troca → por deslocamento | +8,8% | −36,2% |
| Inserção Go: por troca → por deslocamento | −72,6% | não executado |
| Seleção Go: clássica → heapsort | −98,9% | −99,0% |
| Seleção Rust: clássica → heapsort | −99,3% | −99,4% |

Os três pares de constante multiplicativa que rodaram nas duas máquinas mudaram de sinal. A
mesma ideia, o mesmo código-fonte, a mesma lista de entrada, e a economia de 29% da flutuação
com limite em Rust vira um empate de 1,7% em desvantagem no Dell. O deslocamento na inserção
em Rust economiza 36% numa máquina e custa 8,8% na outra, com desvios pequenos dos dois lados,
o que descarta ruído como explicação.

Nossa hipótese, e aqui é hipótese, é que as duas otimizações trocam trabalho aritmético por
trabalho que o processador mais novo já fazia de graça. O laço de limite fixo da versão
ingênua tem contagem de iterações conhecida, o que ajuda o compilador a desenrolar e
vetorizar; o limite móvel da versão otimizada cria uma dependência carregada pelo laço que
atrapalha essa transformação. No Broadwell, onde a versão ingênua não ganha tanto com isso,
cortar comparações continua valendo a pena. No Tiger Lake, o que se economiza em comparações
se perde em código pior gerado. Confirmar exigiria `perf stat -e instructions,cycles` nas
quatro células: se o número de instruções retiradas cair e o tempo não cair junto, a
explicação é essa.

A economia de energia acompanhou a de tempo em todos os seis pares, com uma exceção que vale
seção própria.

### A otimização que aumentou o consumo terminando antes

A flutuação com extração, em Go, é a única configuração em que energia e tempo apontam para
lados opostos, e ela faz isso nas duas máquinas:

| Máquina | Versão | Energia (J) | Tempo (s) | Potência (W) | EDP (J·s) |
|---|---|---|---|---|---|
| Dell | ingênua | 189,121 | 14,525 | 13,02 | 2.747 |
| Dell | com extração | 171,855 | 12,296 | 13,98 | 2.113 |
| HP | ingênua | 118,647 | 19,969 | 5,94 | 2.369 |
| HP | com extração | 124,724 | 19,760 | 6,31 | 2.465 |

O padrão consistente entre as duas máquinas é a potência: a versão com extração puxa 7,4% a
mais de watts no Dell e 6,2% a mais no HP. O que muda é o que acontece com o tempo. No Dell
ela termina 15,3% mais cedo, e o ganho de tempo cobre o aumento de potência com folga, dando
9,1% de economia e um EDP 23% melhor. No HP ela termina 1,0% mais cedo, o aumento de potência
domina, e o resultado é 5,1% a mais de energia com o EDP 4% pior.

A causa da potência maior é o `slices.Insert` na posição 0, que troca comparações por
movimentação de memória em bloco. Nossa leitura é que essa troca sai cara em watts e barata
em segundos: `memmove` é trabalho de alta vazão, mantém as unidades de load/store e o
subsistema de memória ocupados e sobe o consumo instantâneo do pacote, enquanto o laço de
comparação e desvio da versão ingênua gasta mais ciclos por elemento com menos hardware
ativo. Com 8 MiB de L3 e memória mais rápida, o Dell absorve o tráfego extra e ainda sai
ganhando; com 3 MiB, o HP não absorve. Para confirmar bastaria ler o domínio `dram` do RAPL
separadamente, que já está disponível no HP, e ver se os watts extras estão de fato na
memória.

A lição contraria a intuição que a gente tinha antes de medir. Uma otimização correta no
papel pode piorar o consumo quando o custo de memória dela não entra na conta, e o sinal
dessa piora depende da máquina em que o código roda. O mesmo padrão aparece na seleção
clássica em Go, escrita com `slices.Min`, `slices.Index` e `slices.Delete`: três varreduras
por elemento selecionado contra uma da versão em Rust, e uma penalidade de 20% em energia no
Dell.

### O que sobreviveu

Trocar a seleção clássica pelo heapsort economizou entre 98,9% e 99,4% da energia, nas duas
máquinas e nas duas linguagens, e é a única mudança do conjunto que economizou em todas as
combinações. A seleção clássica compara cerca de 5 × 10⁹ pares para n = 100.000; o heapsort
faz uns 3,4 × 10⁶. Cada comparação evitada é uma instrução que não foi buscada, decodificada
e executada, e um acesso à memória que não aconteceu. Ordenar os mesmos 100 mil inteiros
passou de 32,3 J em 2,41 s para 0,215 J em 0,017 s no Dell, e de 28,8 J em 5,00 s para 0,160 J
em 0,025 s no HP.

Traduzindo os dois tipos de decisão para uma unidade utilizável: um serviço que ordenasse um
milhão de listas de 100 mil inteiros por dia no Dell economizaria 52 kWh por dia trocando a
flutuação ingênua em Go pelo heapsort em Rust, e 0,064 kWh por dia mantendo o heapsort e só
migrando de Go para Rust. A primeira decisão tem cerca de 820 vezes a alavancagem da segunda,
e a comparação é generosa com a linguagem, porque o heapsort é justamente onde a vantagem
relativa de Rust é maior.

### Go contra Rust, com ressalva

Rust venceu em 5 das 6 famílias no Dell, por fatores de 1,20× a 3,28× em energia, e em 4 das
5 no HP. A exceção no Dell é a inserção por deslocamento, onde Go gasta 17% menos e é 39%
mais rápido. No HP, a flutuação ingênua dá empate técnico (0,96× com IC 95% [0,94; 1,00]).
Nas duas máquinas os binários Go sustentam potência um pouco maior que os de Rust, 3,8% no
Dell e 6,3% no HP, o que é compatível com o runtime de Go manter threads e coletor de lixo
ativos ao lado do laço de ordenação.

Esse eixo é o mais fraco do trabalho e não deve ser lido como propriedade das linguagens. As
duas versões otimizadas de flutuação implementam ideias diferentes, o toolchain do Dell não
foi registrado, e a versão do Go no HP é desconhecida. A comparação defensável aqui é versão
contra versão dentro da mesma linguagem e da mesma máquina, que é como as seções anteriores
estão organizadas.

## Limitações

Não medimos linha de base ociosa em nenhuma das máquinas, e essa é a lacuna mais séria. Como
o `perf -a` mede o pacote inteiro, cada número é a soma do trabalho do programa com o consumo
da máquina ligada durante a janela. Dos 189,1 J da flutuação ingênua, não sabemos dizer
quanto é cada coisa. A correção é barata e entra no próximo ciclo: 60 s de repouso medidos
antes de cada bloco, e reportar também a energia líquida.

O piso do instrumento, 0,01 J de resolução e ~0,020 J de custo fixo por execução, torna tudo
abaixo de n = 10.000 ininterpretável e ainda embute 12,8% nos números de heapsort em 100.000.
Ordenar em laço dentro do mesmo processo amortizaria a inicialização.

A janela inclui a leitura da entrada e a impressão da saída, um custo O(n) desprezível para os
quadráticos e nada desprezível para o heapsort. Cada resultado vale para uma única permutação
aleatória por tamanho; entradas já ordenadas mudariam completamente as versões com parada
antecipada. O HP não executou `insert_v2.go`, de modo que o par de inserção em Go só tem
medição em uma máquina, e é a lacuna mais fácil de fechar. E duas máquinas não são uma
amostra: mostramos que o ranking pode mudar com a plataforma, sem estimar com que frequência
isso acontece.

## Conclusão

Mudar a classe de complexidade foi a única economia que atravessou a troca de máquina. O
heapsort cortou mais de 98,9% da energia da seleção clássica nas quatro combinações de
linguagem e plataforma, e a conta subestima o ganho, porque o custo fixo de execução pesa
proporcionalmente mais sobre o lado barato.

Nenhuma das otimizações de constante multiplicativa sobreviveu à troca de plataforma com o
mesmo sinal. Três pares rodaram nas duas máquinas e os três mudaram de lado, com variações de
−36% a +8,8% para a mesma ideia e o mesmo código-fonte. Um relatório escrito com uma máquina
só teria afirmado com confiança coisas que o segundo notebook desmente.

O resultado que vamos lembrar daqui a um ano é a flutuação com extração em Go: uma otimização
correta no papel, que terminou mais cedo nas duas máquinas e mesmo assim gastou mais energia
em uma delas, porque trocou comparações por movimentação de memória e subiu a potência em
7%. A realocação do `slices.Insert` não entra em nenhuma contagem de comparações, e entra no
contador RAPL.

---

*As tabelas completas para n = 10.000 e n = 50.000, com energia, tempos, potência e energia
por elemento, estão em [dados do experimento](/dados/).*

*Reprodução: `scripts/implementacao.sh -r 10 -t "10000 50000 100000"` seguido de
`scripts/medicao.sh saida.csv`. É preciso `perf` com acesso a `power/energy-pkg/`, ou seja
`perf_event_paranoid ≤ 0` ou privilégio de root. Tudo está em
[github.com/heitorpita/GoxRust_energy](https://github.com/heitorpita/GoxRust_energy).*

*Referências: Cruz, L., "Green Software Engineering Done Right", material do curso
Sustainable Software Engineering (TU Delft); Pereira, R. et al., "Energy Efficiency across
Programming Languages", SLE 2017; Khan, K. N. et al., "RAPL in Action", ACM ToMPECS, 2018.*
