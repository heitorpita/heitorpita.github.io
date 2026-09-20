---
layout: post
title:  "Medimos a energia de seis algoritmos de ordenação em Go e Rust"
date:   2026-09-20 09:00:00 -0300
categories: energia algoritmos experimento
---

**Heitor Pita e Gabriel Moura**  
Trabalho de Tópicos Avançados em Computação — experimento de consumo energético de algoritmos de ordenação.  
Código, dados e scripts: [github.com/heitorpita/GoxRust_energy](https://github.com/heitorpita/GoxRust_energy)

<style>
.post-content figure { margin: 2em 0; }
.post-content figure img { width: 100%; height: auto; display: block; border: 1px solid #e1e0d9; border-radius: 4px; }
.post-content figcaption { font-size: 0.85em; color: #52514e; margin-top: 0.6em; line-height: 1.5; }
.post-content table { font-size: 0.85em; width: 100%; }
.post-content table td, .post-content table th { padding: 6px 10px; }
</style>

## A pergunta

Big-O conta operações. A conta de luz cobra joules. As duas coisas estão ligadas, mas entre
a operação que você soma no papel e o elétron que sai da tomada existem um compilador, um
runtime, uma hierarquia de cache e um governador de frequência. Nenhum deles aparece no
somatório.

A pergunta que fomos responder era simples: se eu escrevo duas versões do mesmo algoritmo de
ordenação, uma ingênua e outra com uma ideia a mais, quanto isso economiza de energia de
verdade? Implementamos os três métodos pedidos (flutuação, inserção e seleção) em duas
versões cada, em Go e em Rust, e medimos energia, tempo de parede e tempo de CPU de 720
execuções.

O experimento todo consumiu 7.206 J, ou 2,0 Wh, em 20,2 minutos de CPU. Um bubble sort
ingênuo sozinho, sobre 100 mil elementos, gasta 117,9 J. É 737 vezes a energia que o heapsort
usa para ordenar exatamente a mesma lista.

## O cenário

Todas as medições que aparecem aqui saíram de uma máquina só, para que os números sejam
comparáveis entre si. Usamos um segundo notebook (Dell Inspiron 15 3511, Intel Core
i5-1135G7, 8 GiB DDR4-3200) só como ambiente de desenvolvimento, e nenhum dado dele entra nas
tabelas.

| Item | Configuração |
|---|---|
| Processador | Intel Core i5-5200U @ 2,20 GHz (Broadwell-U, 2 núcleos físicos / 4 threads) |
| Cache | L1d 64 KiB · L1i 64 KiB · L2 512 KiB · L3 3 MiB |
| Memória | 11 GiB de RAM, swap de 976 MiB, 1 nó NUMA |
| Frequência | driver `intel_cpufreq`, governador `schedutil`, 0,5–2,7 GHz, turbo ativo, SMT ligado |
| Sistema operacional | Debian GNU/Linux 12 (bookworm), kernel 6.1.0-52-amd64, x86-64 |
| Toolchain | `go` 1.27.1 · `rustc`/`cargo` 1.94.0 · `g++` 12.2.0 · `python3` 3.11.2 |
| Instrumentação | `perf` 6.1.180, `perf_event_paranoid = 0`, 4 eventos de energia disponíveis |
| Domínios RAPL | `intel-rapl:0` (package-0), `:0:0` (core), `:0:1` (uncore), `:0:2` (dram) |

Duas coisas dessa máquina voltam a aparecer nos resultados, então já fica o registro. A
primeira é o governador `schedutil` com turbo ativo: a frequência muda ao longo da própria
medição. A segunda é que os arquivos `energy_uj` dos domínios RAPL não são legíveis por
usuário comum. O contador de energia só sai pela PMU do `perf`, e aqui ele sai sem `sudo`
apenas porque o `perf_event_paranoid` está em 0. Com o valor padrão do Debian a mesma medição
pediria elevação de privilégio, e o script cobre os dois casos.

## O que foi implementado

São seis programas por linguagem: três algoritmos em duas versões cada. Todos leem a lista da
entrada padrão (primeira linha com `n`, segunda com os `n` inteiros) e imprimem o resultado
ordenado, de forma que o custo de entrada e saída seja igual para todo mundo.

Um aviso sobre nomes. No repositório os arquivos são `v1` e `v2`, mas o número não diz o que
mudou, e para a seleção o heapsort é a `v1` em Go e a `v2` em Rust. Foi confuso até para nós,
então daqui em diante cada versão é chamada pelo que ela faz.

| Algoritmo | Linguagem | Arquivo no repositório | Como chamamos aqui |
|---|---|---|---|
| Flutuação | Go | `go/go_algs/bubble_v1.go` | clássica |
| Flutuação | Go | `go/go_algs/bubble_v2.go` | com extração |
| Flutuação | Rust | `rust/rust_algs/bubble_sort_v1.rs` | clássica |
| Flutuação | Rust | `rust/rust_algs/bubble_sort_v2.rs` | com limite |
| Inserção | Go | `go/go_algs/insert_v1.go` | com trocas |
| Inserção | Go | `go/go_algs/insert_v2.go` | com deslocamento |
| Inserção | Rust | `rust/rust_algs/insertion_sort_v1.rs` | com trocas |
| Inserção | Rust | `rust/rust_algs/insertion_sort_v2.rs` | com deslocamento |
| Seleção | Go | `go/go_algs/selection_v1.go` | heapsort (`container/heap`) |
| Seleção | Go | `go/go_algs/selection_v2.go` | clássica |
| Seleção | Rust | `rust/rust_algs/selection_sort_v1.rs` | clássica |
| Seleção | Rust | `rust/rust_algs/selection_sort_v2.rs` | heapsort |

### Flutuação: passadas fixas contra limite móvel

A versão clássica é o bubble sort do livro: dois laços aninhados, `n` passadas completas de
`n-1` comparações cada, troca por variável auxiliar. Ela faz o mesmo trabalho para qualquer
entrada, inclusive para um vetor que já chegue ordenado.

A versão com limite, em Rust, guarda o índice da última troca de cada passada. Tudo que está
à direita desse índice já está no lugar definitivo, então a passada seguinte encolhe até ali.
A classe de complexidade não muda, continua O(n²) no pior caso, mas o número de comparações
reais cai bastante e o algoritmo para sozinho quando o vetor fica ordenado.

A versão com extração, em Go, tentou a mesma ideia por outro caminho: uma flag de "não houve
troca nesta passada" mais a transferência do maior elemento para um vetor de saída, via
`slices.Insert(sorted, 0, ...)`. A intenção era a mesma. O resultado foi o contrário, e essa
acabou sendo a parte do trabalho que mais rendeu explicação.

### Inserção: trocar contra deslocar

A versão com trocas desce a chave posição a posição, trocando com o vizinho. Cada posição
percorrida custa três atribuições em memória (`aux = a[j]; a[j] = a[j-1]; a[j-1] = aux`), e a
condição do laço ainda relê `a[j]`, que acabou de ser escrito.

A versão com deslocamento guarda a chave numa variável local, empurra os elementos maiores
uma casa para a direita (uma atribuição por posição) e grava a chave uma vez só, no fim. O
número de comparações é idêntico nas duas. O que cai, para cerca de um terço, é o número de
escritas.

### Seleção: varredura completa contra heap

A versão clássica procura o mínimo do sufixo a cada iteração: `n(n-1)/2` comparações, sempre,
para qualquer entrada. Em Go ela foi escrita com `slices.Min`, `slices.Index` e
`slices.Delete`, o que faz o vetor ser percorrido umas três vezes por elemento selecionado em
vez de uma.

O heapsort é a variação O(n log n) pedida no enunciado. Constrói um heap em O(n) e depois
extrai o extremo `n` vezes, cada extração custando O(log n) por causa do *sift-down*. Em Rust
é um heapsort in-place clássico; em Go usamos o `container/heap` da biblioteca padrão sobre o
próprio vetor de entrada. Para n = 100.000 a diferença é de uns 5 × 10⁹ comparações contra
3,4 × 10⁶, três ordens de grandeza.

## Como medimos

A medição sai do `perf stat`, lendo o contador RAPL do pacote do processador:

```
perf stat -x';' -a -e power/energy-pkg/,duration_time \
    /usr/bin/time -f '%U;%S' ./binario < lista_100000.in
```

Tem três decisões escondidas nessa linha.

A primeira é o `-a`, que é obrigatório. O contador `power/energy-pkg/` vem da PMU `power`,
que é por pacote e não por tarefa; pedir a energia só do processo devolve `<not supported>`.
A consequência é que o número medido não é "a energia do algoritmo", e sim a energia do
pacote inteiro enquanto o algoritmo roda. Por isso o experimento exige o resto da máquina
ocioso.

A segunda é de onde vem cada tempo. O `duration_time` do `perf` dá o tempo de parede, mas os
eventos `user_time` e `system_time` em modo `-a` somam a máquina inteira. O tempo de CPU do
processo vem então do GNU `time` (`%U` e `%S`), encaixado entre o `perf` e o binário.

A terceira é que a compilação fica de fora. Os binários são gerados antes (`rustc -O -C
debuginfo=0` e `go build`) e só a execução entra no `perf`.

Três scripts em Bash automatizam o resto:

- `scripts/criacao_lista.sh` compila o gerador em C++ (`g++ -O2 -Wall`) e cria as listas de
  10, 100, 1.000, 10.000, 50.000 e 100.000 elementos. O gerador produz a permutação `1..n`
  embaralhada com `2n` trocas aleatórias.
- `scripts/implementacao.sh` compila os fontes, roda cada binário sobre cada lista com o
  `perf` e grava um CSV por (algoritmo, tamanho, execução). Ele checa o `perf_event_paranoid`
  e só eleva privilégio se precisar; quando precisa, renova o *timestamp* do `sudo` entre
  execuções, porque uma ordenação de 20 s estoura o intervalo padrão. Aceita `--repeticoes`,
  `--tamanhos`, `--timeout` e `--limpar`.
- `scripts/consolida.py` junta os CSVs do `perf` num arquivo só, calcula potência média e
  energia por elemento e recalcula médias e desvios de cada configuração. Ao remedir uma
  configuração, ele substitui a célula inteira, nunca misturando execuções de rodadas
  diferentes.

As figuras e a análise deste post saem de `scripts/analise.py`, que lê o CSV consolidado e
gera os SVGs e as tabelas sem nenhuma dependência externa.

Foram 10 execuções por configuração, com a mesma lista de entrada para todos os algoritmos de
um dado tamanho: 12 configurações × 6 tamanhos × 10 = 720 execuções.

Os dados vêm de duas sessões na mesma máquina. As seis configurações de Rust foram medidas na
primeira. As seis de Go foram remedidas do zero em 20/09, numa sessão só, com a máquina
ociosa e o toolchain registrado (`go1.27.1`), depois que descobrimos que a versão de Go usada
na primeira sessão não estava mais instalada e portanto não dava para declarar. Chato, mas a
comparação entre versões de uma mesma linguagem, que é o eixo do trabalho, ficou inteira
dentro de uma sessão só.

## Resultados

### As magnitudes

<figure>
  <a href="{{ '/assets/img/energia/fig1-energia-100k.svg' | relative_url }}"><img src="{{ '/assets/img/energia/fig1-energia-100k.svg' | relative_url }}" alt="Barras horizontais da energia média por configuração em n = 100.000, agrupadas por algoritmo"></a>
  <figcaption>Figura 1 — Energia média por execução em n = 100.000 elementos. Cada painel tem escala própria; a barra preta é um desvio padrão sobre as 10 execuções.</figcaption>
</figure>

| Algoritmo | Linguagem | Versão | Energia (J) | Duração (s) | user (s) | sys (s) | Potência (W) |
|---|---|---|---|---|---|---|---|
| Flutuação | Go | clássica | 117,9 | 19,8 | 19,8 | 0,025 | 5,96 |
| Flutuação | Go | com extração | 120,5 | 19,5 | 19,5 | 0,027 | 6,19 |
| Flutuação | Rust | clássica | 123,0 | 20,8 | 20,8 | 0,008 | 5,90 |
| Flutuação | Rust | com limite | 87,4 | 15,1 | 15,1 | 0,007 | 5,78 |
| Inserção | Go | com trocas | 26,1 | 4,64 | 4,64 | 0,006 | 5,63 |
| Inserção | Go | com deslocamento | 11,7 | 1,81 | 1,80 | 0,003 | 6,49 |
| Inserção | Rust | com trocas | 11,6 | 1,98 | 1,97 | 0,005 | 5,88 |
| Inserção | Rust | com deslocamento | 7,43 | 1,24 | 1,23 | 0,007 | 5,98 |
| Seleção | Go | clássica | 36,4 | 5,92 | 5,92 | 0,013 | 6,14 |
| Seleção | Go | heapsort | 0,338 | 0,050 | 0,043 | 0,001 | 6,82 |
| Seleção | Rust | clássica | 28,8 | 5,00 | 4,99 | 0,005 | 5,75 |
| Seleção | Rust | heapsort | 0,160 | 0,025 | 0,021 | 0,003 | 6,34 |

Repare que o `user_time` é praticamente igual ao `duration_time` em todas as linhas, e que o
`system_time` é ruído. Os programas são monothread e limitados por CPU, sem espera de disco
nem de rede para esconder nada. A única linha em que a diferença aparece é o heapsort em Go,
onde `user + sys` (0,044 s) fica abaixo do tempo de parede (0,050 s): a execução é tão curta
que a subida do runtime de Go pesa mais que a ordenação em si.

A potência média, por outro lado, quase não varia. As 72 configurações ficam todas entre
4,95 W e 7,04 W, com média de 5,93 W para n ≥ 10.000. A Figura 4 mostra o que isso implica.

### O crescimento

<figure>
  <a href="{{ '/assets/img/energia/fig2-escalabilidade.svg' | relative_url }}"><img src="{{ '/assets/img/energia/fig2-escalabilidade.svg' | relative_url }}" alt="Curvas log-log de energia em função do tamanho da entrada, um painel por algoritmo"></a>
  <figcaption>Figura 2 — Energia em função do tamanho da entrada, em escala log-log. A inclinação da reta é o expoente do crescimento.</figcaption>
</figure>

Entre n = 10.000 e n = 100.000, o expoente medido das dez versões quadráticas fica entre 1,92
e 2,02. A teoria aparece na medição de energia com precisão razoável, o que já foi uma boa
surpresa. Os dois heapsorts, porém, ficam em 0,82 e 0,90, bem abaixo de 1, o que parece
contradizer o `n log n`. Não contradiz: é o piso de medição.

O piso é aquela parte plana do lado esquerdo de todos os painéis. Abaixo de n ≈ 1.000 a
energia medida estaciona entre 0,015 e 0,032 J (mediana 0,019 J), que é o custo de subir o
processo, ler a entrada e imprimir a saída, mais o consumo ocioso da máquina durante esses
poucos milissegundos, já que o `perf` mede o pacote inteiro. Para n pequeno, o algoritmo
simplesmente desaparece dentro do arcabouço da medição. Os heapsorts, que em 100.000
elementos ainda gastam só 0,16 a 0,338 J, estão perto demais desse piso para a inclinação
observada valer alguma coisa.

### A reprodutibilidade

<figure>
  <a href="{{ '/assets/img/energia/fig3-boxplot-100k.svg' | relative_url }}"><img src="{{ '/assets/img/energia/fig3-boxplot-100k.svg' | relative_url }}" alt="Boxplots do desvio percentual de cada execução em relação à mediana da sua configuração"></a>
  <figcaption>Figura 3 — Dispersão das 10 execuções de cada configuração em n = 100.000, como desvio percentual da mediana. Em joules as configurações estão a três ordens de grandeza umas das outras e as caixas somem; normalizando, todas ficam comparáveis.</figcaption>
</figure>

O coeficiente de variação das versões quadráticas fica entre 0,29% e 2,5%. A explicação
tentadora seria que execuções mais longas são mais estáveis, mas os dados não ajudam: a
configuração mais demorada, Rust clássica com 20,8 s, é justamente a de maior dispersão
(2,5%), enquanto Go clássica, com 19,8 s, fica em 0,29%. O que separa as duas não é a
duração, é a sessão. Todas as seis linhas de Go vêm da rodada de 20/09, com a máquina
deliberadamente ociosa, e nenhuma passa de 1,7%; as de Rust vêm da primeira sessão e chegam a
2,5%. Dentro do próprio experimento, é a evidência mais direta de que uma medição
*system-wide* mede também o que mais estiver rodando na máquina.

As duas fontes intrínsecas de variação continuam lá, claro. Uma é o `schedutil`, que muda a
frequência ao longo de uma execução de 20 s. A outra é a resolução do contador: o `perf`
reporta energia com duas casas decimais, ou seja, passos de 10 mJ. Numa configuração que
gasta 0,160 J no total, isso dá 6% de granularidade. É exatamente por isso que o heapsort em
Go, com mediana de 0,330 J, tem a caixa mais larga do gráfico, com bigode chegando a +12%,
enquanto o heapsort em Rust, travado em 0,160 J, aparece como uma linha sem largura nenhuma.
As dez execuções caíram todas no mesmo passo do contador.

### Energia é tempo

<figure>
  <a href="{{ '/assets/img/energia/fig4-energia-x-duracao.svg' | relative_url }}"><img src="{{ '/assets/img/energia/fig4-energia-x-duracao.svg' | relative_url }}" alt="Dispersão de energia contra duração para as 720 execuções, em escala log-log, com reta de potência constante"></a>
  <figcaption>Figura 4 — As 720 execuções, energia contra duração, em escala log-log. A reta tracejada é a potência média do pacote.</figcaption>
</figure>

Se der para levar um gráfico só deste trabalho, é esse. As 720 execuções (seis tamanhos, três
algoritmos, duas linguagens, duas versões) caem quase todas sobre uma reta de inclinação 1,
cuja constante é a potência média do pacote. Nesta máquina, energia é tempo multiplicado por
uma constante.

Isso não é uma lei geral. Em processadores com faixa dinâmica de potência maior, ou em cargas
que usam aceleradores, unidades vetoriais ou vários núcleos, o mesmo trabalho pode ser feito
em regimes de potência bem diferentes, e aí a escolha entre "rápido e faminto" e "lento e
econômico" passa a existir de fato. Aqui, com seis programas monothread, inteiros de 32/64
bits e um i5-5200U em `schedutil`, esse grau de liberdade não aparece. Na prática, otimizar
tempo é otimizar energia, e não há trade-off nenhum para decidir.

### O ganho de cada segunda versão

<figure>
  <a href="{{ '/assets/img/energia/fig5-ganho.svg' | relative_url }}"><img src="{{ '/assets/img/energia/fig5-ganho.svg' | relative_url }}" alt="Barras divergentes com a variação percentual de energia da versão melhorada em relação à de partida"></a>
  <figcaption>Figura 5 — Variação percentual da energia média da versão melhorada em relação à versão de partida, em n = 100.000. Negativo é economia.</figcaption>
</figure>

## Por que essas diferenças economizam energia

Com a Figura 4 na mão, explicar cada resultado vira uma conta de trabalho executado.

O heapsort economiza 99,4% em Rust e 99,1% em Go. É o único caso em que a classe de
complexidade muda, e o efeito é de outra ordem. A seleção clássica compara `n(n-1)/2 ≈ 5 ×
10⁹` pares para n = 100.000; o heapsort faz uns `2n log₂ n ≈ 3,4 × 10⁶`. Cada comparação
evitada é uma instrução que não foi buscada, decodificada e executada, e um acesso à memória
que não aconteceu. Em joules: 28,8 J → 0,160 J em Rust, 36,4 J → 0,338 J em Go. Ordenar 100
mil inteiros sai de cinco segundos para 25 milissegundos.

A inserção com deslocamento economiza 55,1% em Go e 36,2% em Rust, e aqui a complexidade é a
mesma nos dois casos, com exatamente o mesmo número de comparações. O que muda é o custo
constante. Trocar dois elementos exige três atribuições e uma releitura do valor
recém-escrito; deslocar exige uma atribuição e mantém a chave em registrador. Menos escritas
significam menos tráfego entre o núcleo e o cache L1, menos pressão no *store buffer* e menos
dependências entre instruções consecutivas, então a CPU consegue manter mais operações em
voo. Um terço das escritas virou 36% de energia a menos em Rust e 55% em Go. A constante
multiplicativa, que a notação O(·) joga fora, é exatamente onde mora essa economia.

Por que o ganho em Go é tão maior que em Rust? Não medimos o porquê, e medir exigiria contar
instruções em vez de joules. A hipótese mais simples é a verificação de limites: cada acesso
indexado a um *slice* em Go carrega um teste que o compilador nem sempre consegue eliminar, e
a versão com trocas faz cinco acessos indexados por posição percorrida contra dois da versão
com deslocamento. Em Rust o otimizador remove boa parte dessas verificações em laços desse
formato, então sobra menos para economizar. O ponto de chegada é curioso: a versão com trocas
em Go gastava 26,1 J e a com deslocamento gasta 11,7 J, praticamente empatada com a versão
com trocas de Rust, que gasta 11,6 J.

A flutuação com limite economiza 29,0%, também sem mudar de classe. Guardar a posição da
última troca faz cada passada encolher para a região que ainda pode estar desordenada, e
sobre uma permutação aleatória isso corta as passadas finais quase inteiras: 20,8 s viram
15,1 s, 123,0 J viram 87,4 J. O custo da otimização é uma variável inteira a mais.

E a flutuação com extração gastou 2,1% a mais. Esse é o contraexemplo. A versão em Go
implementa a mesma ideia, detectar que o vetor já está ordenado, mas transfere cada elemento
para um vetor de saída com `slices.Insert(sorted, 0, ...)`. Inserir na posição 0 de um
*slice* é O(n): todo o conteúdo é deslocado, e a cada crescimento o runtime realoca e copia o
vetor inteiro. O algoritmo passou a fazer um trabalho O(n²) adicional de movimentação de
memória para economizar comparações, e o saldo ficou negativo, 117,9 J contra 120,5 J. A
lição é o oposto da intuição que a gente tinha antes de medir: uma otimização algorítmica
correta no papel pode aumentar o consumo se o custo de memória dela não entrar na conta. O
mesmo padrão aparece na seleção clássica em Go, escrita com `slices.Min` + `slices.Index` +
`slices.Delete`: três varreduras por elemento selecionado contra uma da versão em Rust, e 26%
mais energia (36,4 J contra 28,8 J).

## Limitações

O que este experimento não mostra, ponto a ponto.

A medição é de pacote, não de processo, então o número inclui o consumo ocioso da máquina
durante a execução. Serve para comparar configurações entre si na mesma máquina ociosa, mas
não autoriza dizer "o algoritmo X consome Y joules" em termos absolutos.

Os dados vêm de uma máquina só, com governador `schedutil` e turbo ativo. Uma máquina com
governador fixo em `performance` ou em `powersave` daria outras constantes na Figura 4.
Provavelmente com as mesmas proporções entre algoritmos, mas não testamos.

A comparação entre linguagens é indicativa, não controlada. Go e Rust diferem em runtime,
coletor de lixo, estratégia de *bounds checking* e nível de otimização padrão, e as versões
melhoradas de flutuação nem são a mesma ideia nas duas linguagens. Some a isso os dois blocos
virem de sessões diferentes, Rust da primeira e Go da segunda. O eixo principal do trabalho é
a comparação entre versões da mesma linguagem, cada par inteiramente contido em uma sessão, e
é assim que as figuras estão organizadas.

E a explicação para o ganho maior da inserção em Go é hipótese, não resultado. Confirmar
exigiria contadores de instruções retiradas e de acessos à memória (`perf stat -e
instructions,mem-stores`), o que ficou fora do escopo desta rodada.

## Conclusão

Mudar a classe de complexidade domina todo o resto: o heapsort economiza mais de 99% da
energia da seleção clássica, e nenhuma otimização de constante chega perto disso.

Mas as otimizações de constante não são desprezíveis, e foi isso que mais nos surpreendeu.
Trocar três escritas por uma na inserção rendeu 55% em Go e 36% em Rust; encolher a passada
na flutuação rendeu 29%. Em aplicação real, ganho dessa ordem raramente está disponível em
outro lugar.

O terceiro resultado é o negativo, e é o que a gente vai lembrar daqui a um ano: uma
"melhoria" que ignorou o custo de movimentação de memória piorou o consumo em 2%.

Nesta máquina a regra acabou sendo simples, porque a potência é praticamente constante: menos
trabalho executado, menos tempo; menos tempo, menos joules. A parte interessante é que "menos
trabalho" precisa incluir o trabalho que a linguagem faz por baixo do pano. A realocação do
`slices.Insert` não entra em nenhuma contagem de comparações, mas entra no contador RAPL.

---

*As tabelas completas, com energia, duração, tempo de CPU e potência para os seis tamanhos de
entrada, estão em [dados do experimento](/dados/).*

*Reprodução: todos os scripts, os dados brutos das 720 execuções e o gerador das figuras
estão em [github.com/heitorpita/GoxRust_energy](https://github.com/heitorpita/GoxRust_energy).
O experimento completo roda com `scripts/implementacao.sh -r 10` seguido de
`scripts/consolida.py` e `scripts/analise.py`.*
