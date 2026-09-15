Fazer o Laboratório 1 do SOSIM. 
Responder.
Postar no GitHub PDF do Questionário respondido.
Postar link do Google classroom.



SOsim – Laboratório de Gerência do Processador – Setembro de 2003

Este laboratório cobre os principais conceitos relacionados com a gerência do processador e pode ser obtido em http://www.training.com.br/sosim. 


Exercício 1: Escalonamento circular 
Configurar o escalonamento circular (sem prioridade): janela Gerência do Processador / Opções
Criar dois processos com a mesma prioridade (um CPU-bound e outro I/O-bound)

Na janela Console Sosim / Processos / Selecionar observe o tempo de processador de cada processo durante três minutos e as mudanças de estado. Após esse período anote o tempo de processador de cada processo. Analise a distribuição do uso da UCP.

Pergunta: O que acontece se o tempo de time slice (quantum) aumentar ou diminuir?


Exercício 2: Escalonamento circular com prioridades 

Configurar o escalonamento por prioridades: janela Gerência do Processador / Opções
Criar um processo CPU-bound com prioridade 3 e um I/O-bound com prioridade 4.

Na janela Console Sosim / Processos / Selecionar observe o tempo de processador de cada processo durante três minutos e as mudanças de estado. Após esse período anote o tempo de processador de cada processo. Analise a distribuição do uso da UCP comparativamente ao Exercício 1.

Pergunta: O que acontece se o tempo de espera do processo I/O-bouns aumentar ou diminuir?


Exercício 3: Escalonamento circular com prioridades 

Configurar o escalonamento por prioridades: janela Gerência do Processador / Opções
Criar um processo CPU-bound com prioridade 4 e um I/O-bound com prioridade 3.

Analise a situação.

Pergunta: Quais devem ser os critérios para determinar as prioridades de processos?


Exercício 4: Escalonamento circular com prioridades (prioridade dinâmica – mecanismo adaptivo)

Configurar o escalonamento por prioridades: janela Gerência do Processador / Opções
Criar dois processos com a mesma prioridade (um CPU-bound e outro I/O-bound)

Observe o escalonamento dos processos. Compare a situação desse escalonamento com o apresentado no Exercício 2.

Pergunta: Qual a vantagem desse escalonamento em processos I/O-bound de perfis diferentes?
