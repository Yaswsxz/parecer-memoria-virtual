<div align="center">

# UniCesumar
EDUCAÇÃO PRESENCIAL E A DISTÂNCIA

</div>

---

<div align="center">

YASMIN FERNANDA DE CARVALHO – R.A. 25061121-2

<br><br><br><br><br><br>

**O COLAPSO DO SERVIDOR DE DADOS: MEMÓRIA VIRTUAL, PAGE FAULT E TLB EM UM CASO DE THRASHING**

<br><br><br><br><br><br><br><br>

LONDRINA<br>
2026

</div>

---

<div align="center">

YASMIN FERNANDA DE CARVALHO – R.A. 25061121-2

<br><br><br>

**O COLAPSO DO SERVIDOR DE DADOS: MEMÓRIA VIRTUAL, PAGE FAULT E TLB EM UM CASO DE THRASHING**

</div>

<br><br>

<div align="right">

Desafio Prático: Diagnóstico de Arquitetura<br>
(Encontro 8 – Gerenciamento de Memória),<br>
apresentado ao Curso de Engenharia de Software<br>
para obtenção parcial de nota semestral.

</div>

<br><br><br>

<div align="center">

LONDRINA<br>
2026

</div>

---

## SUMÁRIO

[INTRODUÇÃO](#introdução)

[1. DESENVOLVIMENTO](#1-desenvolvimento)
- [1.1. Parte A: O Fenômeno da Paginação e o Gargalo do Disco](#11-parte-a-o-fenômeno-da-paginação-e-o-gargalo-do-disco)
  - [1.1.1. Memória Virtual, Páginas e Molduras](#111-memória-virtual-páginas-e-molduras)
  - [1.1.2. O que é um Page Fault](#112-o-que-é-um-page-fault)
  - [1.1.3. Thrashing: CPU ociosa e disco sobrecarregado](#113-thrashing-cpu-ociosa-e-disco-sobrecarregado)
- [1.2. Parte B: O Papel e a Eficiência da TLB](#12-parte-b-o-papel-e-a-eficiência-da-tlb)
  - [1.2.1. O que é a TLB e por que fica em hardware](#121-o-que-é-a-tlb-e-por-que-fica-em-hardware)
  - [1.2.2. Impacto da alta taxa de TLB miss](#122-impacto-da-alta-taxa-de-tlb-miss)
  - [1.2.3. A TLB na troca de contexto](#123-a-tlb-na-troca-de-contexto)
- [1.3. Parte C: Síntese e Solução](#13-parte-c-síntese-e-solução)
  - [1.3.1. O efeito de dobrar a memória RAM](#131-o-efeito-de-dobrar-a-memória-ram)
  - [1.3.2. Medidas complementares](#132-medidas-complementares)
- [1.4. Pesquisa Multimídia e Mapa Mental](#14-pesquisa-multimídia-e-mapa-mental)
  - [1.4.1. Fontes consultadas](#141-fontes-consultadas)
  - [1.4.2. Mapa Mental](#142-mapa-mental)

[2. CONCLUSÃO](#2-conclusão)

[REFERÊNCIAS BIBLIOGRÁFICAS](#referências-bibliográficas)

---

## INTRODUÇÃO

Este trabalho analisa um estudo de caso fictício: a empresa de e-commerce TechBuy possui um servidor de banco de dados robusto que, em eventos de alta temporada, apresenta lentidão extrema. A equipe de monitoramento relatou que o processador não está ocupado com regras de negócio, que o uso do disco rígido está em 100% e que o sistema gasta quase todo o seu tempo realizando trocas de contexto. Nesse cenário, o problema não está na capacidade de processamento, mas na forma como a memória está sendo gerenciada.

O objetivo é elaborar um parecer técnico que explique a causa da lentidão e o funcionamento da arquitetura de memória envolvida. Para isso, o trabalho explica o que é um *page fault* e por que o excesso deles leva ao *thrashing*, apresenta o papel da TLB (*Translation Lookaside Buffer*) e o efeito dos seus erros e das trocas de contexto sobre o desempenho, e discute o impacto de dobrar a memória RAM do servidor à luz do conceito de memória virtual.

A justificativa está no impacto prático do tema: falhas de dimensionamento de memória em servidores de banco de dados degradam o tempo de resposta justamente nos momentos de maior demanda, com perda de vendas e de confiança dos clientes. A metodologia adotada foi a pesquisa bibliográfica e multimídia, com base no material da Aula 8, em livros clássicos de sistemas operacionais, em materiais de aula de universidades públicas, em um vídeo sobre thrashing e em um podcast sobre bancos de dados.

---

## 1. DESENVOLVIMENTO

### 1.1. Parte A: O Fenômeno da Paginação e o Gargalo do Disco

#### 1.1.1. Memória Virtual, Páginas e Molduras

A **memória virtual** é a técnica pela qual o sistema operacional oferece a cada processo um espaço de endereçamento próprio e contínuo, independente da quantidade de memória física instalada, usando o disco como extensão da RAM (TANENBAUM; BOS, 2015). Esse espaço virtual é dividido em blocos de tamanho fixo chamados **páginas**, enquanto a memória física é dividida em blocos do mesmo tamanho chamados **molduras** (*frames*). A **MMU** (*Memory Management Unit*) traduz cada endereço virtual em um endereço físico consultando a **tabela de páginas**, na qual cada entrada indica em qual moldura está a página e se ela está presente na RAM (SILBERSCHATZ; GALVIN; GAGNE, 2018; FERRAZ, 2015).

Assim, apenas as páginas que o processo está usando precisam ficar na RAM. As demais ficam na área de troca (*swap*) em disco, e o programa pode ser maior do que a memória física disponível (MACHADO; MAIA, 2013).

#### 1.1.2. O que é um Page Fault

Um ***page fault*** (falta de página) é uma exceção gerada pela MMU quando o processo referencia uma página cujo bit de presença indica que ela não está em nenhuma moldura da RAM (TANENBAUM; BOS, 2015; FERRAZ, 2015). Ao receber a exceção, o sistema operacional executa, em linhas gerais, os seguintes passos (SILBERSCHATZ; GALVIN; GAGNE, 2018):

1. verifica se o endereço referenciado é válido (se não for, o processo é encerrado);
2. procura uma moldura livre; se não houver, escolhe uma página "vítima" por um algoritmo de substituição de páginas e, se ela tiver sido modificada, a grava no disco;
3. dispara a leitura da página necessária do disco para a moldura;
4. atualiza a tabela de páginas e a TLB;
5. reinicia a instrução que causou a falta.

O custo desse mecanismo é muito alto: um acesso à RAM leva dezenas de nanossegundos, enquanto um acesso ao disco leva milissegundos, uma diferença de várias ordens de grandeza. Por isso, mesmo uma pequena taxa de page faults reduz muito o desempenho do sistema (SILBERSCHATZ; GALVIN; GAGNE, 2018).

#### 1.1.3. Thrashing: CPU ociosa e disco sobrecarregado

O quadro descrito no estudo de caso é o de ***thrashing***, situação em que ocorre paginação excessiva e o sistema passa a ser percebido como muito lento ou até parado (GATE SMASHERS, 2018). Em outras palavras, o sistema gasta mais tempo paginando (movendo páginas entre RAM e disco) do que executando os processos (TANENBAUM; BOS, 2015). Ele ocorre quando a soma dos **conjuntos de trabalho** (*working sets*, as páginas que cada processo está usando ativamente) é maior do que a RAM disponível. O encadeamento do problema é o seguinte:

1. Na alta temporada, os processos do banco de dados (conexões, buffers e índices) passam a demandar mais memória do que a RAM comporta.
2. Cada processo, ao executar, referencia páginas que não estão residentes e gera page faults.
3. Para atender cada falta, o sistema expulsa outra página da RAM, muitas vezes de um processo que logo vai precisar dela, o que gera novo page fault em seguida.
4. Todo processo que sofre um page fault fica **bloqueado aguardando a E/S do disco**. Sem processos prontos para executar, a **CPU fica ociosa**.
5. As leituras de páginas necessárias e as gravações de páginas modificadas formam uma fila crescente de requisições, e o **disco chega a 100% de uso**.
6. Como os processos bloqueiam o tempo todo, o escalonador realiza **trocas de contexto** em excesso, sem que nenhum trabalho útil seja concluído.

Há ainda um efeito de realimentação: se o sistema interpretar a CPU ociosa como baixa carga e admitir mais processos, a RAM disponível por processo diminui e o thrashing se agrava (SILBERSCHATZ; GALVIN; GAGNE, 2018).

### 1.2. Parte B: O Papel e a Eficiência da TLB

#### 1.2.1. O que é a TLB e por que fica em hardware

A **TLB** (*Translation Lookaside Buffer*) é um cache pequeno e muito rápido, localizado na MMU, que guarda as traduções de página virtual para moldura usadas recentemente, junto com os bits de proteção e de modificação. Ela é uma memória **associativa**: compara o número da página virtual com todas as suas entradas ao mesmo tempo (TANENBAUM; BOS, 2015; FERRAZ, 2015).

Ela é implementada em hardware porque a tradução de endereços ocorre **em toda referência à memória**, tanto na busca de instruções quanto nos acessos a dados. Se a tabela de páginas ficasse apenas na RAM, cada acesso exigiria pelo menos dois acessos à memória (um à tabela e outro ao dado), e mais ainda em tabelas de vários níveis. Uma tabela comum na RAM seria lenta demais para esse caminho crítico. A TLB funciona bem por causa da **localidade de referência**: os programas concentram a maior parte dos acessos em poucas páginas (MACHADO; MAIA, 2013).

#### 1.2.2. Impacto da alta taxa de TLB miss

Quando a tradução é encontrada na TLB (*hit*), o custo é o de uma consulta rápida mais o acesso ao dado. Quando não é encontrada (*miss*), é preciso percorrer a tabela de páginas na RAM, o que acrescenta um ou mais acessos à memória antes de chegar ao dado. O **tempo efetivo de acesso** pode ser expresso por:

> EAT = h · (t_TLB + t_mem) + (1 − h) · (t_TLB + n · t_mem + t_mem)

em que *h* é a taxa de acerto da TLB, *t_TLB* é o tempo de consulta à TLB, *t_mem* é o tempo de um acesso à memória e *n* é o número de níveis da tabela de páginas (SILBERSCHATZ; GALVIN; GAGNE, 2018). Quanto menor o valor de *h*, maior o tempo médio de cada acesso.

No servidor da TechBuy, um banco de dados que acessa grandes volumes de dados de forma dispersa tem baixa localidade e sofre muitos TLB misses. Esses misses acontecem mesmo quando a página está na RAM, portanto somam-se ao custo dos page faults e aumentam o tempo de resposta de cada consulta.

#### 1.2.3. A TLB na troca de contexto

Cada processo possui seu próprio espaço de endereçamento, então a mesma página virtual aponta para molduras diferentes em processos diferentes. Por isso, na troca de contexto as traduções guardadas na TLB para o processo anterior deixam de ser válidas para o novo processo (MACHADO; MAIA, 2013). Há duas soluções:

1. **Invalidar (*flush*) a TLB** a cada troca. O novo processo começa com a TLB vazia e sofre uma sequência de misses até reconstruí-la.
2. **Identificar o espaço de endereços em cada entrada** (como o ASID ou PCID), o que permite manter traduções de vários processos na TLB sem esvaziá-la (TANENBAUM; BOS, 2015).

Com as trocas de contexto frequentes do cenário estudado, a invalidação repetida mantém a TLB sempre "fria", o que aumenta a taxa de TLB miss e piora o tempo de resposta.

### 1.3. Parte C: Síntese e Solução

#### 1.3.1. O efeito de dobrar a memória RAM

Dobrar a RAM física **reduz de forma significativa** a taxa de page faults e o uso do disco, desde que a causa do problema seja a RAM ser menor do que o conjunto de trabalho total dos processos, como indica o diagnóstico. A justificativa baseia-se no próprio conceito de memória virtual:

- a memória virtual permite que o espaço de endereçamento total seja maior do que a RAM, usando o disco como extensão, mas isso só é eficiente enquanto os conjuntos de trabalho dos processos cabem na memória física;
- com mais molduras, mais páginas permanecem residentes, e o número de substituições e de leituras e gravações de página no disco diminui;
- se o conjunto de trabalho total passar a caber na RAM, o thrashing desaparece: o disco volta a ser usado apenas para a E/S legítima do banco, a CPU volta a ter processos prontos e as trocas de contexto diminuem (SILBERSCHATZ; GALVIN; GAGNE, 2018).

Existem duas ressalvas. Primeiro, se mesmo com o dobro de RAM o conjunto de trabalho total ainda a superar, o thrashing continua, apenas em outro patamar de carga. Segundo, mais RAM não elimina por si só os TLB misses, pois a TLB tem tamanho fixo. Nesse ponto ajudam as páginas grandes (*huge pages*), que aumentam a quantidade de memória coberta por cada entrada da TLB (TANENBAUM; BOS, 2015).

#### 1.3.2. Medidas complementares

Além de ampliar a memória, recomendam-se as seguintes medidas:

- limitar o número de conexões e processos simultâneos, controlando o grau de multiprogramação, uma das formas de prevenir o thrashing (GATE SMASHERS, 2018);
- ajustar os *buffers* do banco de dados ao tamanho da RAM disponível;
- usar o conceito de conjunto de trabalho, ou a frequência de page faults, para controlar a quantidade de memória alocada a cada processo (SILBERSCHATZ; GALVIN; GAGNE, 2018);
- revisar consultas e índices que causam varreduras amplas e destroem a localidade de referência;
- usar SSD ou NVMe na área de troca apenas como medida paliativa, pois isso não trata a causa.

### 1.4. Pesquisa Multimídia e Mapa Mental

#### 1.4.1. Fontes consultadas

Para embasar o parecer, foram cruzadas informações de três formatos de fonte:

- **Vídeo:** a aula "L-5.16: What is Thrashing", do canal Gate Smashers (2018), apresenta o thrashing como a condição em que ocorrem operações excessivas de paginação, passando pelos conceitos de paginação e de page fault e por formas de prevenir o problema. O vídeo foi usado para sustentar a definição de thrashing e as medidas de prevenção da seção 1.3.2.
- **Áudio:** o episódio #404 do podcast Hipsters Ponto Tech, "SQL e MySQL" (2024), discute o dia a dia de quem trabalha com bancos de dados. Ele foi usado para contextualizar o papel do servidor de banco de dados no caso da TechBuy, enquanto a relação com o gerenciamento de memória foi construída a partir das demais fontes.
- **Texto:** Tanenbaum e Bos (2015), Silberschatz, Galvin e Gagne (2018), Machado e Maia (2013) e os slides de Ferraz (2015), usados para os conceitos de memória virtual, page fault, TLB e conjunto de trabalho.

#### 1.4.2. Mapa Mental

O mapa mental abaixo relaciona os conceitos de memória virtual, page fault, TLB e disco rígido, conectando-os ao que foi lido, ouvido e assistido.

![Mapa mental](https://github.com/Yaswsxz/O-Gargalo-da-Mem-ria-Virtual/blob/main/mapa-mental.png?raw=true)
---

## 2. CONCLUSÃO

O objetivo do trabalho foi alcançado: a partir dos conceitos de memória virtual, paginação e TLB, foi possível explicar o comportamento do servidor da TechBuy e propor uma solução para a lentidão relatada.

A pesquisa mostrou que o quadro de CPU ociosa, disco a 100% e excesso de trocas de contexto corresponde ao *thrashing*: a soma dos conjuntos de trabalho dos processos supera a RAM, o que gera page faults em cadeia e faz os processos passarem quase todo o tempo bloqueados esperando o disco. A TLB, por sua vez, é essencial para que a tradução de endereços seja rápida, mas sofre com a baixa localidade dos acessos do banco e com o esvaziamento a cada troca de contexto, o que agrava a lentidão.

A metodologia de pesquisa bibliográfica e multimídia foi suficiente para responder ao problema. Dobrar a memória RAM é uma solução adequada, pois permite que os conjuntos de trabalho caibam na memória física e elimina o principal gerador de page faults. Para que o resultado seja duradouro, ela deve ser combinada com o controle do número de processos simultâneos, o ajuste dos buffers do banco e a otimização de consultas.

---

## REFERÊNCIAS BIBLIOGRÁFICAS

FERRAZ, Carlos. **Sistemas operacionais**: gerenciamento de memória: memória virtual. Recife: CIn-UFPE, 2015. Slides de aula. Disponível em: https://cin.ufpe.br/~jcbf/if677/2015-1/slides/Aula_10_Memoria_Virtual.pdf. Acesso em: 18 out. 2026.

<br><br>

GATE SMASHERS. **L-5.16**: What is Thrashing | Operating System. [*S. l.*]: YouTube, 11 maio 2018. 1 vídeo. Disponível em: https://www.youtube.com/watch?v=IyWaK8pbN6A. Acesso em: 18 out. 2026.

<br><br>

HIPSTERS PONTO TECH. **SQL e MySQL**: Hipsters Ponto Tech #404. Apresentador: Paulo Silveira. [*S. l.*]: Alura, 2 abr. 2024. Podcast. Disponível em: https://www.hipsters.tech/sql-e-mysql-hipsters-ponto-tech-404/. Acesso em: 18 out. 2026.

<br><br>

MACHADO, Francis Berenger; MAIA, Luiz Paulo. **Arquitetura de sistemas operacionais**. 5. ed. Rio de Janeiro: LTC, 2013.

<br><br>

SILBERSCHATZ, Abraham; GALVIN, Peter Baer; GAGNE, Greg. **Fundamentos de sistemas operacionais**. 9. ed. Rio de Janeiro: LTC, 2018.

<br><br>

TANENBAUM, Andrew S.; BOS, Herbert. **Sistemas operacionais modernos**. 4. ed. São Paulo: Pearson, 2015.
