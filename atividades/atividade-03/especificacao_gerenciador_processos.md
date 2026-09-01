# Atividade 03 — Especificação do Gerenciador de Processos de um Simulador de Sistema Operacional

**Disciplina:** Sistemas Operacionais  
**Equipe:** Izadora de Oliveira Albuquerque Montenegro, João Guilherme Santos de Sousa e Karen Menezes Ribeiro

---

## 1. Visão Geral e Arquitetura do Simulador

### 1.1 Objetivo

O projeto consiste na especificação de um simulador de gerenciamento de processos capaz de representar, de forma simplificada, as principais responsabilidades de um sistema operacional relacionadas à criação, execução, bloqueio, escalonamento e término de processos.

O simulador será executado inteiramente em **modo usuário** e não utilizará processos ou threads reais do sistema operacional hospedeiro para representar os processos simulados. CPU, interrupções, operações de entrada/saída, estados e passagem do tempo serão representados internamente por estruturas de dados e eventos controlados pelo próprio simulador.

O simulador deverá permitir observar:

- criação e término de processos;
- alternância entre os estados dos processos;
- utilização da CPU;
- bloqueio causado por operações de entrada/saída;
- interrupções causadas pelo término do quantum;
- trocas de contexto;
- funcionamento de diferentes algoritmos de escalonamento;
- impacto dos algoritmos no tempo de espera, tempo de resposta e tempo de retorno.

Cada processo será considerado **single-threaded**, isto é, possuirá um único fluxo de execução. O escalonamento será realizado diretamente sobre processos.

### 1.2 Componentes principais

O simulador será dividido conceitualmente nos seguintes componentes:

| Componente | Responsabilidade |
| --- | --- |
| CPU Virtual | Representar a execução do processo atualmente selecionado. |
| Relógio Lógico | Representar a passagem do tempo através de ticks. |
| Núcleo Simulado | Coordenar eventos, estados, CPU, E/S e escalonamento. |
| PCB | Armazenar o contexto e as informações de cada processo. |
| Tabela de Processos | Armazenar todos os PCBs existentes. |
| Gerenciador de E/S | Controlar processos bloqueados e a duração das operações de E/S. |
| Escalonador | Selecionar qual processo pronto deverá receber a CPU. |
| Leitor de Tarefas | Interpretar o arquivo utilizado como entrada da simulação. |
| Gerador de Relatórios | Produzir logs, gráfico de Gantt e estatísticas. |

A separação entre núcleo e escalonador deverá permitir que diferentes algoritmos de escalonamento sejam utilizados sem modificar a lógica principal da simulação.

### 1.3 CPU virtual

A CPU simulada deverá possuir os seguintes elementos:

| Campo | Descrição |
| --- | --- |
| `pc` | Contador de programa do processo atualmente em execução. |
| `registradores` | Conjunto simplificado de registradores de propósito geral. |
| `processo_atual` | PID do processo utilizando a CPU ou valor nulo caso ela esteja ociosa. |
| `quantum_restante` | Quantidade de ticks restantes do quantum atual, quando aplicável. |

Serão considerados quatro registradores de propósito geral:

```text
R0
R1
R2
R3
```

Os registradores não precisam representar instruções reais. Sua função é permitir a simulação do salvamento e da restauração de contexto durante trocas de processo.

O contador de programa (`pc`) será incrementado a cada tick de CPU consumido pelo processo.

### 1.4 Relógio lógico

A passagem do tempo será representada por um relógio lógico discreto.

A simulação começará com:

```text
clock = 0
```

Cada iteração completa do laço principal corresponde a **um tick**.

Exemplo:

```text
clock = 0
clock = 1
clock = 2
clock = 3
...
```

Um tick representa uma unidade abstrata de tempo e será utilizado para calcular todas as durações e estatísticas.

### 1.5 Fluxo geral de execução

Antes da execução, o simulador deverá receber:

1. o arquivo de tarefas;
2. o algoritmo de escalonamento;
3. os parâmetros específicos do algoritmo, quando necessários.

Após validar a entrada, o simulador iniciará o relógio lógico.

Em cada tick, deverá:

1. verificar processos cuja chegada esteja programada para o instante atual;
2. criar esses processos e inseri-los na fila de prontos;
3. atualizar operações de E/S em andamento;
4. liberar processos cuja E/S tenha terminado;
5. verificar se existe necessidade de preempção por prioridade;
6. selecionar um processo caso a CPU esteja livre;
7. executar um tick da rajada de CPU do processo atual;
8. atualizar o `pc`, registradores simulados, tempos e quantum;
9. verificar término de rajada, término de processo ou expiração do quantum;
10. realizar troca de contexto quando necessário;
11. registrar os acontecimentos relevantes no log;
12. atualizar tempos de espera e de bloqueio;
13. incrementar o relógio lógico.

A simulação terminará quando todos os processos definidos na entrada atingirem o estado `TERMINADO`.

---

## 2. Especificação do Bloco de Controle de Processo (PCB) e Tabela de Processos

### 2.1 Estados possíveis

Cada processo deverá estar em exatamente um dos seguintes estados:

```text
NOVO
PRONTO
EXECUTANDO
BLOQUEADO
TERMINADO
```

Os três estados operacionais principais são:

- `PRONTO`: apto a executar, aguardando CPU;
- `EXECUTANDO`: utilizando a CPU virtual;
- `BLOQUEADO`: aguardando conclusão de uma operação de E/S.

Os estados `NOVO` e `TERMINADO` delimitam o início e o fim do ciclo de vida.

### 2.2 Estrutura do PCB

Cada processo será representado por exatamente um PCB.

| Campo | Tipo conceitual | Descrição |
| --- | --- | --- |
| `pid` | inteiro | Identificador único do processo. |
| `estado` | enum | Estado atual do processo. |
| `prioridade_base` | inteiro | Prioridade definida na entrada. |
| `prioridade_atual` | inteiro | Prioridade considerada pelo escalonador após possíveis alterações por aging. |
| `pc` | inteiro | Contador de programa salvo. |
| `registradores` | vetor de inteiros | Valores salvos dos registradores da CPU virtual. |
| `tempo_chegada` | inteiro | Tick em que o processo entra no sistema. |
| `tempo_inicio` | inteiro ou nulo | Primeiro tick em que o processo recebe CPU. |
| `tempo_termino` | inteiro ou nulo | Tick em que o processo termina. |
| `tempo_cpu` | inteiro | Total de ticks em estado `EXECUTANDO`. |
| `tempo_espera` | inteiro | Total de ticks em estado `PRONTO`. |
| `tempo_bloqueado` | inteiro | Total de ticks em estado `BLOQUEADO`. |
| `operacoes` | lista | Sequência de rajadas de CPU e E/S. |
| `operacao_atual` | inteiro | Índice da operação atualmente em processamento. |
| `tempo_restante` | inteiro | Quantidade de ticks restantes da operação atual. |
| `tempo_espera_prioridade` | inteiro | Contador usado para o mecanismo de aging. |

O `pid` deverá ser único durante toda a execução.

### 2.3 Representação das operações

Cada operação deverá possuir:

```text
tipo
duracao
```

O tipo poderá assumir apenas:

```text
CPU
IO
```

Exemplo:

```text
CPU 4
IO 3
CPU 2
```

Esse programa representa:

1. utilização da CPU durante 4 ticks;
2. bloqueio por E/S durante 3 ticks;
3. utilização da CPU durante 2 ticks;
4. término do processo.

Uma sequência de operações deverá obrigatoriamente:

- começar com `CPU`;
- alternar entre `CPU` e `IO`;
- terminar com `CPU`;
- possuir duração maior que zero em todas as operações.

### 2.4 Tabela de processos

O núcleo manterá uma tabela de processos indexada por PID.

Exemplo conceitual:

```text
TabelaProcessos
    1 -> PCB(P1)
    2 -> PCB(P2)
    3 -> PCB(P3)
```

O PCB armazenado nessa tabela será a **fonte única de verdade** sobre o estado de cada processo.

Filas e estruturas auxiliares deverão armazenar somente PIDs ou referências para os PCBs, evitando cópias divergentes da mesma informação.

A tabela deverá permitir, no mínimo:

```text
criar_processo(...)
obter_processo(pid)
listar_processos()
listar_processos_por_estado(estado)
atualizar_processo(pid)
```

Processos terminados deverão permanecer disponíveis até o fim da simulação para permitir a geração das estatísticas finais.

---

## 3. Ciclo de Vida e Grafo de Transição de Estados

### 3.1 Grafo geral

```text
                    criação
                      |
                      v
                    NOVO
                      |
                      v
                   PRONTO <------------------+
                      |                      |
                      | despacho             | E/S concluída
                      v                      |
                 EXECUTANDO ------------> BLOQUEADO
                    |   |
                    |   | solicitação de E/S
                    |
        quantum     |
        expirado    +---------------------> PRONTO
                    |
                    | última rajada concluída
                    v
                 TERMINADO
```

### 3.2 NOVO → PRONTO

A criação ocorrerá quando:

```text
clock == tempo_chegada
```

O núcleo deverá:

1. criar o PCB;
2. registrar o processo na tabela;
3. definir `estado = NOVO`;
4. inicializar os contadores;
5. carregar a primeira rajada;
6. alterar para `PRONTO`;
7. inserir o processo na estrutura de prontos do escalonador.

Essa operação representa de forma simplificada a criação de processo, equivalente conceitualmente a uma operação como `fork`.

### 3.3 PRONTO → EXECUTANDO

Quando a CPU estiver livre e existir pelo menos um processo pronto:

1. o escalonador selecionará o próximo PID;
2. o processo será removido da estrutura de prontos;
3. seu estado mudará para `EXECUTANDO`;
4. seu `pc` e registradores serão restaurados;
5. `processo_atual` receberá seu PID;
6. o quantum será inicializado, quando aplicável.

Na primeira vez em que o processo receber CPU:

```text
tempo_inicio = clock
```

### 3.4 EXECUTANDO → PRONTO por quantum

Essa transição ocorre no Round Robin quando:

```text
quantum_restante == 0
```

e a rajada atual ainda não terminou.

O núcleo deverá:

1. salvar `pc` e registradores no PCB;
2. alterar o estado para `PRONTO`;
3. reinserir o PID no final da fila;
4. limpar `processo_atual`;
5. liberar a CPU;
6. permitir novo despacho.

### 3.5 EXECUTANDO → BLOQUEADO

Quando uma rajada de CPU terminar e existir uma próxima operação do tipo `IO`:

1. salvar o contexto;
2. avançar `operacao_atual`;
3. carregar a duração da E/S em `tempo_restante`;
4. alterar o estado para `BLOQUEADO`;
5. inserir o processo na estrutura de E/S;
6. liberar a CPU.

Processos bloqueados nunca poderão ser escolhidos pelo escalonador.

### 3.6 BLOQUEADO → PRONTO

Todas as operações de E/S poderão avançar simultaneamente.

A cada tick, para cada processo bloqueado:

```text
tempo_restante = tempo_restante - 1
tempo_bloqueado = tempo_bloqueado + 1
```

Quando:

```text
tempo_restante == 0
```

o simulador deverá:

1. considerar a E/S concluída;
2. avançar `operacao_atual`;
3. carregar a próxima rajada de CPU;
4. alterar o estado para `PRONTO`;
5. inserir o processo na estrutura do escalonador.

A E/S não utiliza a CPU virtual.

### 3.7 EXECUTANDO → TERMINADO

Quando a rajada de CPU terminar e não existir outra operação:

```text
estado = TERMINADO
tempo_termino = clock
```

O processo não retornará à fila de prontos.

A CPU será liberada e outro processo poderá ser despachado.

### 3.8 Troca de contexto

Toda substituição do processo atualmente em execução deverá gerar uma troca de contexto.

Ao retirar um processo da CPU:

```text
CPU.pc -> PCB.pc
CPU.registradores -> PCB.registradores
```

Ao colocar um novo processo:

```text
PCB.pc -> CPU.pc
PCB.registradores -> CPU.registradores
```

Para simplificar o modelo, a troca de contexto terá **custo zero** e não consumirá ticks adicionais.

---

## 4. Especificação do Escalonador de CPU

### 4.1 Interface comum

Todos os algoritmos de escalonamento deverão implementar um contrato equivalente a:

```text
adicionar(pid)
remover(pid)
selecionar_proximo() -> pid
atualizar_tick()
```

O núcleo deverá depender somente dessa interface comum.

A escolha do algoritmo deverá ser feita por configuração, por exemplo:

```text
--scheduler=rr
```

ou:

```text
--scheduler=priority
```

---

### 4.2 Round Robin

O Round Robin utilizará uma fila FIFO.

Exemplo:

```text
[P1, P2, P3]
```

Com quantum igual a 2:

```text
quantum = 2
```

P1 poderá executar no máximo dois ticks consecutivos.

Caso ainda tenha CPU restante após o quantum:

```text
[P2, P3, P1]
```

#### Regras

1. processos novos entram no final da fila;
2. processos liberados de E/S entram no final da fila;
3. processos interrompidos por quantum voltam ao final da fila;
4. o primeiro processo da fila é sempre o próximo escolhido;
5. o quantum deve ser inteiro e positivo;
6. se a rajada de CPU terminar exatamente junto com o quantum, a conclusão da rajada terá prioridade sobre a preempção.

Exemplo de configuração:

```text
--scheduler=rr --quantum=3
```

---

### 4.3 Escalonamento por prioridade

Cada processo possuirá prioridade entre:

```text
0 e 9
```

A convenção adotada será:

```text
0 = prioridade mais alta
9 = prioridade mais baixa
```

Exemplo:

```text
P1 prioridade 4
P2 prioridade 1
P3 prioridade 7
```

A ordem inicial de escolha será:

```text
P2 -> P1 -> P3
```

### 4.4 Desempate

Quando dois ou mais processos possuírem a mesma prioridade:

1. terá preferência o processo que estiver esperando há mais tempo;
2. persistindo empate, terá preferência o menor PID.

Essas regras garantem determinismo.

### 4.5 Preempção por prioridade

O escalonamento por prioridade será **preemptivo**.

Se um processo estiver executando e outro entrar em `PRONTO` com prioridade maior, deverá ocorrer preempção.

Exemplo:

```text
P1 EXECUTANDO prioridade 5
P2 PRONTO prioridade 2
```

Como:

```text
2 < 5
```

P2 deverá assumir a CPU.

Resultado:

```text
P1: EXECUTANDO -> PRONTO
P2: PRONTO -> EXECUTANDO
```

### 4.6 Prevenção de starvation com aging

Para impedir inanição, será utilizado aging.

Cada processo pronto manterá:

```text
tempo_espera_prioridade
```

A cada tick em `PRONTO`:

```text
tempo_espera_prioridade += 1
```

A cada cinco ticks consecutivos de espera:

```text
prioridade_atual -= 1
```

respeitando o limite:

```text
prioridade_atual >= 0
```

Exemplo:

```text
Prioridade inicial: 7

5 ticks esperando  -> 6
10 ticks esperando -> 5
15 ticks esperando -> 4
```

Quando o processo for despachado:

```text
prioridade_atual = prioridade_base
tempo_espera_prioridade = 0
```

Assim, a melhoria temporária de prioridade vale somente durante aquele período de espera.

---

## 5. Arquivo de Tarefas

### 5.1 Formato de entrada

O simulador deverá ler um arquivo textual simples.

Cada linha válida representa um processo no seguinte formato:

```text
PID PRIORIDADE CHEGADA OPERACAO OPERACAO ...
```

Exemplo:

```text
P1 3 0 CPU:5 IO:3 CPU:2
P2 1 0 CPU:3 IO:2 CPU:4
P3 5 2 CPU:8
```

Interpretação:

```text
P1
prioridade = 3
chegada = 0
CPU por 5 ticks
IO por 3 ticks
CPU por 2 ticks
```

Linhas iniciadas por `#` serão comentários e deverão ser ignoradas.

Exemplo:

```text
# pid prioridade chegada rajadas
P1 3 0 CPU:5 IO:3 CPU:2
P2 1 1 CPU:4
```

### 5.2 Validação da entrada

O simulador deverá rejeitar o arquivo caso:

- exista PID repetido;
- o PID esteja ausente;
- a prioridade esteja fora do intervalo `0..9`;
- o tempo de chegada seja negativo;
- não existam operações;
- uma duração seja menor ou igual a zero;
- a primeira operação não seja `CPU`;
- a última operação não seja `CPU`;
- existam duas operações consecutivas do mesmo tipo;
- exista um tipo diferente de `CPU` ou `IO`.

A execução não deverá iniciar caso a entrada seja inválida.

O erro deverá informar a linha e o motivo.

Exemplo:

```text
Erro na linha 4: prioridade 12 fora do intervalo permitido (0..9).
```

---

## 6. Saída do Simulador

### 6.1 Log de transições

Todas as mudanças de estado deverão ser registradas.

Formato:

```text
[tick=0] P1: NOVO -> PRONTO
[tick=0] P1: PRONTO -> EXECUTANDO
[tick=2] P2: NOVO -> PRONTO
[tick=3] P1: EXECUTANDO -> PRONTO (quantum expirado)
[tick=3] P2: PRONTO -> EXECUTANDO
```

Eventos de E/S deverão incluir informações adicionais:

```text
[tick=7] P2: EXECUTANDO -> BLOQUEADO (IO, duracao=3)
[tick=10] P2: BLOQUEADO -> PRONTO (IO concluida)
```

Eventos de término:

```text
[tick=15] P1: EXECUTANDO -> TERMINADO
```

### 6.2 Gráfico de Gantt

O simulador deverá gerar um Gantt textual mostrando os intervalos de utilização da CPU.

Exemplo:

```text
0       3       5       8
|  P1   |  P2   |  P1   |
```

Se a CPU ficar sem processos prontos:

```text
IDLE
```

Exemplo:

```text
0       2           6       7
|  P1   |   IDLE    |  P1   |
```

Intervalos consecutivos do mesmo processo poderão ser agrupados.

---

## 7. Estatísticas

### 7.1 Estatísticas por processo

Para cada processo, deverão ser calculados:

#### Tempo de retorno

```text
turnaround = tempo_termino - tempo_chegada
```

#### Tempo de resposta

```text
response_time = tempo_inicio - tempo_chegada
```

#### Tempo de espera

Total de ticks no estado `PRONTO`.

#### Tempo de CPU

Total de ticks no estado `EXECUTANDO`.

#### Tempo bloqueado

Total de ticks no estado `BLOQUEADO`.

Exemplo:

```text
PID | CPU | Espera | Bloqueado | Resposta | Turnaround
P1  |  7  |   4    |     3     |    0     |    14
P2  |  4  |   2    |     0     |    1     |     6
```

### 7.2 Estatísticas globais

#### Utilização da CPU

```text
utilizacao_cpu =
ticks_cpu_ocupada / ticks_totais * 100
```

#### Throughput

```text
throughput =
numero_processos_terminados / tempo_total
```

Também deverão ser calculados:

- tempo médio de espera;
- tempo médio de resposta;
- tempo médio de retorno;
- número total de trocas de contexto;
- tempo total da simulação.

Exemplo:

```text
=====================================
SIMULACAO CONCLUIDA
=====================================

Escalonador: Round Robin
Quantum: 3
Tempo total: 24 ticks
Processos concluídos: 4
Trocas de contexto: 9

Utilização da CPU: 87.5%
Throughput: 0.167 processos/tick
Tempo médio de espera: 5.25 ticks
Tempo médio de resposta: 1.75 ticks
Tempo médio de retorno: 13.50 ticks
```

---

## 8. Casos de Teste

### 8.1 Teste 1 — Processo único sem E/S

Entrada:

```text
P1 3 0 CPU:5
```

Resultado esperado:

```text
NOVO -> PRONTO -> EXECUTANDO -> TERMINADO
```

A CPU deverá permanecer ocupada por cinco ticks.

Objetivo: validar o ciclo básico de vida.

### 8.2 Teste 2 — Round Robin

Configuração:

```text
quantum = 2
```

Entrada:

```text
P1 3 0 CPU:4
P2 3 0 CPU:3
```

Sequência esperada:

```text
0-2 P1
2-4 P2
4-6 P1
6-7 P2
```

Gantt:

```text
0       2       4       6   7
|  P1   |  P2   |  P1   |P2|
```

Objetivo: validar quantum e reinserção no final da fila.

### 8.3 Teste 3 — Operação de E/S

Entrada:

```text
P1 3 0 CPU:2 IO:3 CPU:2
P2 3 0 CPU:4
```

Após os dois primeiros ticks de CPU:

```text
P1: EXECUTANDO -> BLOQUEADO
```

Enquanto P1 estiver bloqueado, P2 poderá utilizar a CPU.

Após três ticks de E/S:

```text
P1: BLOQUEADO -> PRONTO
```

Objetivo: validar bloqueio e retorno da E/S.

### 8.4 Teste 4 — Escalonamento por prioridade

Entrada:

```text
P1 5 0 CPU:3
P2 2 0 CPU:3
P3 7 0 CPU:3
```

A primeira seleção deverá ser:

```text
P2
```

Objetivo: validar a ordenação por prioridade.

### 8.5 Teste 5 — Preempção por prioridade

Entrada:

```text
P1 5 0 CPU:8
P2 1 2 CPU:2
```

Em `tick=0`, P1 inicia.

Em `tick=2`, P2 chega.

Como P2 possui prioridade maior:

```text
P1: EXECUTANDO -> PRONTO
P2: PRONTO -> EXECUTANDO
```

Objetivo: validar preempção por chegada de processo mais prioritário.

### 8.6 Teste 6 — Aging

Entrada com um processo de baixa prioridade concorrendo com processos de prioridade superior.

Exemplo:

```text
P1 8 0 CPU:5
P2 1 0 CPU:15
P3 2 1 CPU:10
```

Durante o tempo em que P1 permanecer em `PRONTO`, sua prioridade deverá melhorar a cada cinco ticks.

Objetivo: validar prevenção de starvation.

### 8.7 Teste 7 — CPU ociosa

Entrada:

```text
P1 3 0 CPU:2 IO:4 CPU:1
```

Gantt esperado:

```text
0       2           6   7
|  P1   |   IDLE    |P1|
```

Objetivo: validar representação de períodos sem processos prontos.

### 8.8 Teste 8 — Chegada tardia

Entrada:

```text
P1 3 5 CPU:3
```

Entre `tick=0` e `tick=5` a CPU deverá estar `IDLE`.

O processo somente poderá entrar em `PRONTO` no tick 5.

Objetivo: validar `tempo_chegada`.

### 8.9 Teste 9 — Entrada inválida

Entrada:

```text
P1 15 0 IO:-2 CPU:3
```

A execução deverá ser recusada devido a:

- prioridade fora do intervalo;
- duração negativa;
- primeira operação diferente de CPU.

Objetivo: validar tratamento de erros de entrada.

---

## 9. Regras de Determinismo

A mesma entrada, o mesmo algoritmo e os mesmos parâmetros deverão produzir sempre exatamente a mesma simulação.

Nenhuma decisão poderá depender de:

- velocidade da máquina;
- threads reais;
- processos reais;
- relógio físico;
- ordem não determinística de estruturas internas.

Quando múltiplos eventos ocorrerem no mesmo tick, a ordem será:

1. registrar chegadas;
2. concluir operações de E/S;
3. verificar preempção por prioridade causada por processos recém-prontos;
4. despachar um processo se a CPU estiver livre;
5. executar um tick da CPU;
6. atualizar tempos e registradores;
7. verificar término da rajada;
8. verificar término do processo;
9. verificar solicitação de E/S;
10. verificar expiração do quantum;
11. atualizar tempos de espera dos processos prontos;
12. aplicar aging;
13. registrar o estado final do tick;
14. incrementar `clock`.

Se a rajada de CPU e o quantum terminarem simultaneamente, a conclusão da rajada terá prioridade sobre a interrupção de relógio.

---

## 10. Critérios de Encerramento

A simulação será encerrada quando:

```text
todos os processos da entrada estiverem em TERMINADO
```

Ao final, deverão ser exibidos:

1. algoritmo utilizado;
2. parâmetros utilizados;
3. log completo de transições;
4. gráfico de Gantt;
5. estatísticas individuais;
6. estatísticas globais.

---

## 11. Diretrizes de Implementação

A implementação gerada a partir desta especificação deverá manter separadas as responsabilidades dos seguintes módulos:

```text
CPU
Nucleo
PCB
TabelaDeProcessos
GerenciadorDeIO
Escalonador
RoundRobin
EscalonadorPrioridade
LeitorDeTarefas
GeradorDeRelatorios
```

Os algoritmos Round Robin e Prioridade deverão seguir uma interface comum.

O núcleo não deverá depender diretamente dos detalhes internos de um algoritmo específico.

A implementação não deverá utilizar processos ou threads reais para representar os processos simulados.

Toda concorrência deverá ser simulada pelo relógio lógico.

O programa deverá priorizar comportamento determinístico e clareza da simulação.

---

## 12. Diretrizes de Entrega

A especificação deverá ser entregue em formato Markdown (`.md`) e publicada no GitHub de cada integrante da equipe.

Este arquivo deverá funcionar como entrada para posterior geração e implementação do simulador por meio de um Harness de desenvolvimento assistido por IA.

Qualquer comportamento não explicitamente definido deverá preservar os seguintes princípios:

- execução determinística;
- separação entre núcleo e algoritmos de escalonamento;
- PCB como fonte única de verdade sobre cada processo;
- ausência de concorrência real;
- passagem do tempo exclusivamente por relógio lógico;
- respeito ao ciclo de estados especificado;
- suporte intercambiável a Round Robin e Prioridade;
- geração obrigatória de logs, Gantt e estatísticas.
