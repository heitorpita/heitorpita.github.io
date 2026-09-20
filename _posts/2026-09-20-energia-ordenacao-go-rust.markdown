---
layout: post
title:  "Dois jeitos de ordenar, duas contas de luz: medindo energia de algoritmos em Go e Rust"
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

Complexidade assintótica é uma conversa sobre contagem de operações. Conta de luz é uma
conversa sobre joules. As duas estão ligadas, mas não são a mesma coisa: entre a operação
contada no papel e o elétron gasto no processador existem um compilador, um runtime, uma
hierarquia de cache e um governador de frequência.

A pergunta deste trabalho é direta: **se eu escrever duas versões do mesmo algoritmo de
ordenação, uma ingênua e outra com uma ideia a mais, quanta energia isso economiza de
verdade?** Implementamos os três métodos pedidos — flutuação (bubble sort), inserção
(insertion sort) e seleção (selection sort) — em duas versões cada, em **Go** e em **Rust**,
e medimos energia, tempo de parede e tempo de CPU de **720 execuções**.

O experimento inteiro consumiu 7.206 J (2,0 Wh) em 20,2 minutos de CPU. Um único bubble sort
ingênuo sobre 100 mil elementos consome 117,9 J — **737 vezes** a energia que o heapsort gasta
para ordenar exatamente a mesma lista.

## O cenário

Todas as medições reportadas aqui foram feitas em uma única máquina, para que os números
sejam comparáveis entre si. O grupo usou um segundo notebook (Dell Inspiron 15 3511, Intel
Core i5-1135G7, 8 GiB DDR4-3200) apenas como ambiente de desenvolvimento; nenhum dado dele
entra nas tabelas.

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

Duas características desse hardware aparecem nos resultados mais adiante e vale registrá-las
desde já: o governador é `schedutil` com turbo ativo, ou seja, **a frequência varia durante a
medição**; e os arquivos `energy_uj` dos domínios RAPL não são legíveis por usuário comum.
O contador de energia só é acessível pela PMU do `perf` — e, nesta máquina, sem elevação de
privilégio apenas porque `perf_event_paranoid` está em **0**. Com o valor padrão do Debian a
mesma medição exigiria `sudo`, e o script trata os dois casos.

## O que foi implementado

São seis programas por linguagem: três algoritmos × duas versões. Todos leem a lista da
entrada padrão (primeira linha com `n`, segunda com os `n` inteiros) e imprimem o resultado
ordenado, de modo que o custo de entrada/saída seja o mesmo para todos.

Uma observação sobre a nomenclatura: no repositório os arquivos são `v1` e `v2`, mas o número
da versão não diz o que mudou — e, no caso da seleção, o heapsort é a `v1` em Go e a `v2` em
Rust. Por isso, daqui em diante, cada versão é chamada pelo que ela faz.

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

### Flutuação: passadas fixas vs. limite móvel

A versão **clássica** é o bubble sort do livro: dois laços aninhados, `n` passadas completas
de `n-1` comparações cada, troca via variável auxiliar. Ela faz o mesmo trabalho
independentemente da entrada — inclusive se o vetor já chegar ordenado.

A versão **com limite** (Rust) guarda o índice da última troca de cada passada. Tudo o que
está à direita desse índice já está no lugar definitivo, então o limite da passada seguinte
encolhe para lá. Não muda a classe de complexidade (continua O(n²) no pior caso), mas corta
uma fatia grande das comparações reais e faz o algoritmo terminar sozinho quando o vetor
fica ordenado.

A versão **com extração** (Go) tentou a mesma ideia por outro caminho: uma *flag* de "não
houve troca nesta passada" e a transferência do maior elemento para um vetor de saída usando
`slices.Insert(sorted, 0, ...)`. A intenção era boa; o efeito, como veremos, foi o contrário
do esperado — e é o resultado mais instrutivo do trabalho.

### Inserção: trocar vs. deslocar

A versão **com trocas** desce a chave posição a posição trocando com o vizinho. Cada posição
percorrida custa três atribuições em memória (`aux = a[j]; a[j] = a[j-1]; a[j-1] = aux`) e a
condição do laço relê `a[j]`, que acabou de ser escrito.

A versão **com deslocamento** guarda a chave em uma variável local, desloca os elementos
maiores uma casa à direita (uma atribuição por posição) e grava a chave uma única vez, no
final. O número de **comparações é idêntico**; o que muda é o número de **escritas**, que cai
para cerca de um terço.

### Seleção: varredura completa vs. heap

A versão **clássica** procura o mínimo do sufixo a cada iteração: `n(n-1)/2` comparações,
sempre, para qualquer entrada. Em Go, essa versão foi escrita com `slices.Min`,
`slices.Index` e `slices.Delete`, o que a faz percorrer o vetor cerca de três vezes por
elemento selecionado, em vez de uma.

A versão **heapsort** é a variação O(n log n) pedida no enunciado. Ela constrói um heap em
O(n) e depois extrai o extremo `n` vezes, cada extração custando O(log n) por causa do
*sift-down*. Em Rust isso é um heapsort in-place clássico; em Go, usa-se o `container/heap`
da biblioteca padrão sobre o próprio vetor de entrada. Para n = 100.000, a diferença é de
cerca de 5 × 10⁹ comparações contra 3,4 × 10⁶ — três ordens de grandeza.

## Como medimos

A medição é feita por `perf stat`, lendo o contador **RAPL** do pacote do processador:

```
perf stat -x';' -a -e power/energy-pkg/,duration_time \
    /usr/bin/time -f '%U;%S' ./binario < lista_100000.in
```

Três decisões de método se escondem nessa linha:

1. **`-a` (system-wide) é obrigatório.** O contador `power/energy-pkg/` vem da PMU `power`,
   que é por pacote e não por tarefa: pedir a energia só do processo devolve
   `<not supported>`. A consequência é que o número medido não é "a energia do algoritmo",
   e sim **a energia do pacote inteiro durante a execução do algoritmo** — por isso o
   experimento roda com o resto da máquina ocioso.
2. **`duration_time` do `perf` dá o tempo de parede**, mas os eventos `user_time` e
   `system_time` do `perf` em modo `-a` somam a máquina inteira. O tempo de CPU do processo
   vem então do **GNU `time`** (`%U` e `%S`), encaixado entre o `perf` e o binário.
3. **A compilação fica fora da medição.** Os binários são gerados antes (`rustc -O -C
   debuginfo=0` e `go build`), e só a execução entra no `perf`.

Todo o experimento é automatizado por três scripts em Bash:

- `scripts/criacao_lista.sh` compila o gerador em C++ (`g++ -O2 -Wall`) e cria as listas de
  10, 100, 1.000, 10.000, 50.000 e 100.000 elementos. O gerador produz a permutação
  `1..n` embaralhada com `2n` trocas aleatórias.
- `scripts/implementacao.sh` compila os fontes, roda cada binário sobre cada lista com o
  `perf` e grava um CSV por (algoritmo, tamanho, execução). Ele checa o
  `perf_event_paranoid` e só eleva privilégio se for necessário — quando é, renova o
  *timestamp* do `sudo` entre execuções, porque uma ordenação de 20 s estoura o intervalo
  padrão. Aceita `--repeticoes`, `--tamanhos`, `--timeout` e `--limpar`.
- `scripts/consolida.py` junta os CSVs do `perf` em um único arquivo, calcula potência média
  e energia por elemento e recalcula as médias e desvios de cada configuração. Ao remedir
  uma configuração, ele a substitui por inteiro — nunca mistura execuções de rodadas
  diferentes dentro da mesma célula.

A análise e as figuras deste post saem de `scripts/analise.py`, que lê o CSV consolidado e
gera os SVGs e as tabelas sem nenhuma dependência externa.

Foram **10 execuções por configuração**, com a mesma lista de entrada para todos os
algoritmos de um dado tamanho. Total: 12 configurações × 6 tamanhos × 10 = 720 execuções.

Os dados vêm de duas sessões de medição na mesma máquina. As seis configurações de **Rust**
foram medidas na primeira sessão; as seis de **Go** foram remedidas do zero em 20/09, em uma
única sessão com a máquina ociosa e com o toolchain registrado (`go1.27.1`), depois que
constatamos que a versão de Go da primeira sessão não estava mais instalada e portanto não
podia ser declarada. A comparação entre versões de uma mesma linguagem, que é o eixo do
trabalho, está inteiramente contida em uma sessão só.

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
| Flutuação | Rust | com limite | **87,4** | 15,1 | 15,1 | 0,007 | 5,78 |
| Inserção | Go | com trocas | 26,1 | 4,64 | 4,64 | 0,006 | 5,63 |
| Inserção | Go | com deslocamento | **11,7** | 1,81 | 1,80 | 0,003 | 6,49 |
| Inserção | Rust | com trocas | 11,6 | 1,98 | 1,97 | 0,005 | 5,88 |
| Inserção | Rust | com deslocamento | **7,43** | 1,24 | 1,23 | 0,007 | 5,98 |
| Seleção | Go | clássica | 36,4 | 5,92 | 5,92 | 0,013 | 6,14 |
| Seleção | Go | heapsort | **0,338** | 0,050 | 0,043 | 0,001 | 6,82 |
| Seleção | Rust | clássica | 28,8 | 5,00 | 4,99 | 0,005 | 5,75 |
| Seleção | Rust | heapsort | **0,160** | 0,025 | 0,021 | 0,003 | 6,34 |

Duas leituras saltam da tabela. A primeira: **`user_time` é praticamente igual ao
`duration_time` em todas as linhas, e `system_time` é ruído**. Os programas são
monothread e limitados por CPU; não há espera de disco nem de rede para esconder. A única
linha em que a diferença é visível é o heapsort em Go, onde `user + sys` (0,044 s) fica
abaixo do tempo de parede (0,050 s): a execução é tão curta que a subida do runtime de Go
pesa mais que a ordenação em si.

A segunda: **a potência média quase não varia**. Todas as 72 configurações ficam entre
4,95 W e 7,04 W, com média de 5,93 W para n ≥ 10.000. Isso tem uma consequência prática
forte, que a Figura 4 deixa explícita.

### O crescimento

<figure>
  <a href="{{ '/assets/img/energia/fig2-escalabilidade.svg' | relative_url }}"><img src="{{ '/assets/img/energia/fig2-escalabilidade.svg' | relative_url }}" alt="Curvas log-log de energia em função do tamanho da entrada, um painel por algoritmo"></a>
  <figcaption>Figura 2 — Energia em função do tamanho da entrada, em escala log-log. A inclinação da reta é o expoente do crescimento.</figcaption>
</figure>

Entre n = 10.000 e n = 100.000, o expoente medido das dez versões quadráticas fica entre
**1,92 e 2,02** — a teoria aparece na medição de energia com precisão razoável. Os dois
heapsorts ficam em **0,82 e 0,90**, bem abaixo de 1, o que parece contradizer o `n log n`: é o
efeito do piso de medição.

Esse piso é a parte plana do lado esquerdo de todos os painéis. Abaixo de n ≈ 1.000, a
energia medida estaciona entre 0,015 e 0,032 J (mediana **0,019 J**), que é o custo de subir o
processo, ler a entrada e imprimir a saída — o `perf` está medindo o pacote inteiro,
inclusive o consumo ocioso da máquina durante esses poucos milissegundos. Para n pequeno, o
algoritmo é invisível dentro do próprio arcabouço da medição. Os heapsorts, que em 100.000
elementos ainda gastam apenas 0,16–0,338 J, estão perto demais desse piso para que a
inclinação observada seja confiável.

### A reprodutibilidade

<figure>
  <a href="{{ '/assets/img/energia/fig3-boxplot-100k.svg' | relative_url }}"><img src="{{ '/assets/img/energia/fig3-boxplot-100k.svg' | relative_url }}" alt="Boxplots do desvio percentual de cada execução em relação à mediana da sua configuração"></a>
  <figcaption>Figura 3 — Dispersão das 10 execuções de cada configuração em n = 100.000, expressa como desvio percentual da mediana. Em joules as configurações estão a três ordens de grandeza umas das outras e as caixas somem; normalizando, todas ficam comparáveis.</figcaption>
</figure>

O coeficiente de variação das versões quadráticas fica entre 0,29% e 2,5%. Tentador dizer
que as execuções mais longas são as mais estáveis — mas os dados não sustentam isso: a
configuração de maior duração, `Rust · clássica` com 20,8 s, é justamente a de maior
dispersão (2,5%), enquanto `Go · clássica`, com 19,8 s, fica em 0,29%. **O que separa as
duas não é a duração, é a sessão de medição.** Todas as seis linhas de Go vêm da rodada de
20/09, com a máquina deliberadamente ociosa, e nenhuma passa de 1,7%; as de Rust vêm da
primeira sessão e chegam a 2,5%. É a evidência mais direta, dentro do próprio experimento,
de que uma medição *system-wide* mede também o que mais estiver rodando na máquina.

Dito isso, as duas fontes de variação intrínsecas continuam visíveis: o governador
`schedutil`, que muda a frequência ao longo de uma execução de 20 s, e a própria resolução do
contador: o `perf` reporta a energia com duas casas decimais, ou seja,
passos de **10 mJ**. Em uma configuração que gasta 0,160 J no total, isso é 6% de
granularidade — e é exatamente por isso que o heapsort em Go, com mediana de 0,330 J, mostra
a caixa mais larga do gráfico, com bigode chegando a +12%, enquanto o heapsort em Rust,
travado em 0,160 J, aparece como uma linha sem largura nenhuma: todas as dez execuções caíram
no mesmo passo do contador.

### Energia é tempo

<figure>
  <a href="{{ '/assets/img/energia/fig4-energia-x-duracao.svg' | relative_url }}"><img src="{{ '/assets/img/energia/fig4-energia-x-duracao.svg' | relative_url }}" alt="Dispersão de energia contra duração para as 720 execuções, em escala log-log, com reta de potência constante"></a>
  <figcaption>Figura 4 — As 720 execuções, energia contra duração, em escala log-log. A reta tracejada é a potência média do pacote.</figcaption>
</figure>

Este é o gráfico que resume o experimento. As 720 execuções — seis tamanhos, três
algoritmos, duas linguagens, duas versões — caem praticamente todas sobre uma reta de
inclinação 1, cuja constante é a potência média do pacote. **Neste hardware, energia é tempo
multiplicado por uma constante.**

Isso não é uma lei geral: em processadores com faixa dinâmica de potência maior, ou em cargas
que usam aceleradores, unidades vetoriais ou vários núcleos, o mesmo trabalho pode ser feito
em regimes de potência bem diferentes, e aí a escolha entre "rápido e faminto" e "lento e
econômico" passa a existir. Aqui, com seis programas monothread, inteiros de 32/64 bits e um
i5-5200U em `schedutil`, esse grau de liberdade simplesmente não aparece. A conclusão prática
é reconfortante: **otimizar o tempo é otimizar a energia**.

### O ganho de cada segunda versão

<figure>
  <a href="{{ '/assets/img/energia/fig5-ganho.svg' | relative_url }}"><img src="{{ '/assets/img/energia/fig5-ganho.svg' | relative_url }}" alt="Barras divergentes com a variação percentual de energia da versão melhorada em relação à de partida"></a>
  <figcaption>Figura 5 — Variação percentual da energia média da versão melhorada em relação à versão de partida, em n = 100.000. Negativo é economia.</figcaption>
</figure>

## Por que essas diferenças economizam energia

Com a Figura 4 na mão, a explicação de cada resultado vira uma conta de trabalho executado.

**Heapsort: −99,4% (Rust) e −99,1% (Go).** É o único caso em que a classe de complexidade
muda, e o efeito é de outra ordem. A seleção clássica compara `n(n-1)/2 ≈ 5 × 10⁹` pares
para n = 100.000; o heapsort faz cerca de `2n log₂ n ≈ 3,4 × 10⁶`. Cada comparação evitada
é uma instrução que não foi buscada, decodificada e executada, e um acesso à memória que não
aconteceu. Em joules: 28,8 J → 0,160 J em Rust, 36,4 J → 0,338 J em Go. Ordenar 100 mil
inteiros passa de cinco segundos para 25 milissegundos.

**Inserção com deslocamento: −55,1% em Go e −36,2% em Rust.** Aqui a complexidade é a mesma
— O(n²) nos dois casos, com exatamente o mesmo número de comparações. O que muda é o custo
constante: trocar dois elementos exige três atribuições e uma releitura do valor
recém-escrito, enquanto deslocar exige uma atribuição e mantém a chave em registrador. Menos
escritas significam menos tráfego entre o núcleo e o cache L1, menos pressão no *store
buffer* e menos dependências entre instruções consecutivas — a CPU consegue manter mais
operações em voo. Um terço das escritas virou 36% de energia a menos em Rust e 55% em Go.
**A constante multiplicativa, que a notação O(·) descarta, é exatamente onde mora essa
economia.**

Por que o ganho em Go é tão maior que em Rust? Não medimos o porquê — isso exigiria contar
instruções, não joules —, mas a hipótese mais simples é a verificação de limites: cada acesso
indexado a um *slice* em Go carrega um teste que o compilador nem sempre consegue eliminar, e
a versão com trocas faz cinco acessos indexados por posição percorrida contra dois da versão
com deslocamento. Em Rust, o otimizador remove boa parte dessas verificações em laços dessa
forma, então sobra menos para economizar. Vale notar o ponto de chegada: a versão com trocas
em Go gastava 26,1 J e a com deslocamento gasta 11,7 J — praticamente empatada com a versão
com trocas de Rust (11,6 J).

**Flutuação com limite: −29,0%.** Também sem mudar a classe. Guardar a posição da última
troca faz cada passada encolher para a região que ainda pode estar desordenada. Sobre uma
permutação aleatória, isso corta as passadas finais quase inteiras: 20,8 s caem para 15,1 s,
e 123,0 J caem para 87,4 J. O custo dessa otimização é uma variável inteira a mais.

**Flutuação com extração: +2,1%.** O contraexemplo. A versão em Go implementa a mesma ideia
— detectar que o vetor já está ordenado —, mas transfere cada elemento para um vetor de saída
com `slices.Insert(sorted, 0, ...)`. Inserir na posição 0 de um *slice* é uma operação O(n):
todo o conteúdo é deslocado, e a cada crescimento o runtime realoca e copia o vetor inteiro.
O algoritmo passou a fazer um trabalho O(n²) *adicional* em movimentação de memória para
economizar comparações, e o saldo ficou negativo: 117,9 J → 120,5 J. Vale registrar a lição,
porque ela é o oposto da intuição corrente: **uma otimização algorítmica correta no papel pode
aumentar o consumo se o seu custo de memória não for contabilizado.** O mesmo padrão aparece
na seleção clássica em Go, escrita com `slices.Min` + `slices.Index` + `slices.Delete`: três
varreduras por elemento selecionado, contra uma da versão em Rust, e 26% mais energia
(36,4 J contra 28,8 J).

## Limitações

Vale ser explícito sobre o que este experimento **não** mostra.

A medição é de pacote, não de processo: o número inclui o consumo ocioso da máquina durante a
execução. Isso é aceitável para comparar configurações entre si na mesma máquina ociosa, mas
não permite dizer "o algoritmo X consome Y joules" em termos absolutos.

Os dados vêm de **uma única máquina**, com governador `schedutil` e turbo ativo. Uma máquina
com governador fixo em `performance` ou em `powersave` daria constantes diferentes na
Figura 4 — provavelmente com as mesmas proporções entre algoritmos, mas isso não foi testado.

A comparação **entre linguagens** é indicativa, não controlada. Go e Rust diferem em runtime,
coletor de lixo, estratégia de *bounds checking* e nível de otimização padrão; além disso,
as versões "melhoradas" de flutuação não são a mesma ideia nas duas linguagens. Soma-se a
isso o fato de os dois blocos virem de sessões diferentes: Rust da primeira, Go da segunda.
O eixo principal do trabalho é a comparação **entre versões da mesma linguagem**, cada par
inteiramente contido em uma sessão, e é assim que as figuras estão organizadas.

Por fim, a explicação para o ganho maior da inserção em Go é uma hipótese, não um resultado.
Confirmá-la exigiria contadores de instruções retiradas e de acessos à memória — `perf stat
-e instructions,mem-stores` —, o que ficou fora do escopo desta rodada.

## Conclusão

Três resultados, em ordem de tamanho do efeito.

Mudar a classe de complexidade domina tudo o mais: o heapsort economiza mais de 99% da
energia da seleção clássica, e nenhuma otimização de constante chega perto disso. Mas as
otimizações de constante **não são desprezíveis**: trocar três escritas por uma na inserção
rendeu 55% em Go e 36% em Rust, e encolher a passada na flutuação rendeu 29% — em aplicações
reais, ganhos dessa ordem raramente estão disponíveis em outro lugar. E o terceiro resultado
é o negativo: uma "melhoria" que ignora o custo de movimentação de memória piorou o consumo
em 2%.

Neste hardware a regra é simples, porque a potência é praticamente constante: **menos
trabalho executado, menos tempo; menos tempo, menos joules.** A parte interessante é que
"menos trabalho" precisa incluir o trabalho que a linguagem faz por baixo do pano — a
realocação do `slices.Insert` não aparece em nenhuma contagem de comparações, mas aparece
com clareza no contador RAPL.

---

*As tabelas completas — energia, duração, tempo de CPU e potência para os seis tamanhos de
entrada — estão em [dados do experimento](/dados/).*

*Reprodução: todos os scripts, os dados brutos das 720 execuções e o gerador das figuras
estão em [github.com/heitorpita/GoxRust_energy](https://github.com/heitorpita/GoxRust_energy).
O experimento completo roda com `scripts/implementacao.sh -r 10` seguido de
`scripts/consolida.py` e `scripts/analise.py`.*
