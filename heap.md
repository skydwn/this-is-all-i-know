# Heap Overflow & GOT Hooking
Antes de falar sobre Heap Overflow, é necessário saber alguns tópicos antes, vou dar uma breve introdução a cada um desses <br>tópicos e falar sobre a exploração de fato.
<br><br>
Quais são esses tópicos?

Alocação dinâmica com malloc e free.<br>
* Global Offset Table x Procedure Linkage Table<br>
* C Memory Layout<br>
* É importante ao menos conhecer o conceito de cada um desses tópicos, porque ambos vão ser aplicados ao decorrer do paper.<br>
<br>
Aqui uma breve introdução a alocação dinâmica em C, usando malloc e free. O malloc é uma função cujo o programador pode indicar um tamanho de um buffer<br>
que vai ser alocado na memória (heap), e pode ser usado em conjunto com outras funções para fazer uma alocação dinâmica de um certo tipo de dado,<br>
por exemplo. Já a função free, apenas libera esse espaço reservado pelo malloc da memória (heap). Porém, atenção a uma coisa, o free não limpa aquela<br>
região da memória, ele apenas libera o espaço, e assim ele pode acabar sendo sobrescrito por qualquer junk na memória. Veja um exemplo de alocação dinâmica. (Fonte: UFU)<br><br>

<img src="https://miro.medium.com/v2/resize:fit:628/format:webp/1*Cn2OyR_DN4kFFIPQeE8OWA.png" width="40%">
<br><br>

Agora vamos falar um pouco de GOT x PLT. Para explicar isso, é necessário que você tenha um pouco de entendimento sobre arquitetura de computadores. 
<br>Nesse tópico, vamos abordar o que é a tabela PLT, e o que é a tabela GOT. (Recomendado entender os conceitos de PIC/PIE)<br><br>

A GOT (Global Offset Table) enumera os symbols de um elf e resolve um absolute memory address em tempo de execução, que posteriormente <br>
será randomizado pelo ASLR.<br>
<br>
Já a PLT (Procedure Linkage Table) resolve os offsets guardados pela GOT e resolve o endereço atual de uma instrução<br> usando o seu respectivo offset.
<br><br>
Agora que falamos sobre GOT x PLT, vamos falar sobre Memory Layout em C. Veja o esquema abaixo:<br><br>

<img src="https://miro.medium.com/v2/resize:fit:640/format:webp/1*b3l-cBj4C1tII5Qs0mEvrw.png" width="40%"><br><br>

Você pode ver nos “lower address”, as sections .text, .data e .bss. Nelas vão ser alocadas as intruções, variáveis, buffers, <br>
entre outros. E também podemos ver as estruturas de dados “heap” e “stack”.<br>
A estrutura de dados “heap” é usada para alocações dinâmicas e prioridade de entrada e saída de dados. Já a estrutura de dados “stack”,<br>
é uma pilha de dados, onde o primeiro dado a entrar, é o primeiro a sair, os dados que saem da stack são direcionados ao instruction pointer (ip).<br><br>

Agora que foram introduzidos a estes tópicos, vamos falar sobre a exploração. Vamos explorar o challenge “heap1” da protostar. Vejam o cenário:<br>
<br><img src="https://miro.medium.com/v2/resize:fit:640/format:webp/1*sJaORv7jgDhKto7kkE8YWw.png" width="40%"><br><br>

Basicamente, temos uma função “winner” que não é chamada em momento algum. Uma alocação dinâmica sendo feita com malloc, e uma cópia com strcpy. 
<br>Para quem já está familiarizado com buffer overflow, sabe que é uma função vulnerável :)<br><br>

Vamos começar então a exploração.<br>

É importante ressaltar uma coisa, em uma exploração stack-based, o atacante só precisa sobrescrever o endereço que vai retornar ao ip<br>
assim podendo alterar o flow do binário. Porém, em heap-based não temos a stack ao nosso favor, então como obteríamos o controle do ip?<br><br>

<img src="https://miro.medium.com/v2/resize:fit:640/format:webp/1*J0SZdI3V38jAtZCeTBXWUw.png" width="40%"><br><br>

Observe o dump da função puts@plt, sabemos que em tempo de execução, a PLT resolve os endereços da GOT, então a instrução “jmp” aponta para um 
<br>endereço resolvido da GOT, puts@got.plt. Você pode confirmar isso usando o “examine” do gdb. Sabendo isso, podemos prosseguir com a exploração.<br><br>

Sabemos que a função “strcpy” é vulnerável, pois ela faz a cópia de dados sem checar o tamanho desse dado, portanto, você pode correr o risco de <br>
sofrer um buffer overflow ao usar essa função. Para confirmar isso, vamos enviar uma grande quantidade de bytes e esperar um “segmentation fault”.<br><br>

<img src="https://miro.medium.com/v2/resize:fit:720/format:webp/1*WrtPxRK2BwCsOdxXbqzLqw.png" width="40%"><br><br>

Como o binário faz a alocação com dois argumentos, é importante que você mantenha esse padrão, para não crashar o binário.<br><br>

Por que deu segmentation fault?<br><br>

Essa é uma pergunta simples de responder. Com base no Memory Layout, você pode notar que após alguns segmentos, 
<br>onde ficam alocados os buffers e afins, vem as estruturas de dados. Portanto, quando você envia mais bytes que um buffer pode suportar, 
<br>ele causa um overflow e começa a sobrescrever outros segmentos da memória com bytes sem sentido, e o binário se perde e da erro.<br><br>

Mas afinal, para onde vai os bytes que excedem o buffer?<br><br>

Simples, leia o manual do strcpy e você vai ver que ele recebe 2 entradas, o “dest” e o “src”, como o próprio nome diz, 
<br>o destino e o valor a ser copiado. Portanto, os bytes sobrescrevem esses valores.<br><br>

Depois de algumas tentativas, você pode descobrir exatamente incrementando e decrementando bytes, com quantos bytes sobrescreve o ponteiro<br>
*src do strcpy();<br><br>

<img src="https://miro.medium.com/v2/resize:fit:720/format:webp/1*WrtPxRK2BwCsOdxXbqzLqw.png" width="40%"><br><br>

Ok, com 24 bytes acontece um overflow, o que significa que se você decrementar 4 bytes (32 bits) <br>
os próximos 4 bytes vai ser alocados no “dest” do strcpy.<br><br>

Como vamos usar isso para tomar o controle do ip?<br>

Vamos pegar o endereço da puts@got.plt<br><br>

<img src="https://miro.medium.com/v2/resize:fit:640/format:webp/1*sDtidEXxOZDwbJKaKhD95w.png" width="40%"><br><br>

Agora, vamos enviar o endereço da puts@got.plt para o “dest” do strcpy, e um junk no segundo strcpy, e ver o que acontece.<br><br>

<img src="https://miro.medium.com/v2/resize:fit:720/format:webp/1*Fyeh-fHaFXluzH67eXY3Vw.png" width="40%"><br><br>

E conseguimos o controle do ip :)<br>
Isso acontece, porque quando a puts@plt é chamada, a PLT resolve o endereço da puts@got.plt. Quando você coloca a 
<br>puts@got.plt como “dest” de uma cópia, você pode passar um argumento que vai ser copiado para puts@got.plt no “src”, e quando 
<br>a puts@plt for chamada, o seu argumento do “src” vai ser chamado :)<br><br>

Agora ficou fácil, vamos pegar o endereço da winner e passar como “src”.<br>

Veja:<br><br>
<img src="https://miro.medium.com/v2/resize:fit:720/format:webp/1*qEi0Fyz1iDkwBxUH6lHMvg.png" width="50%"><br><br>

Espero que tenham gostado e aprendido algo<br>Este artigo/e-zine foi escrito em 02/2021.
