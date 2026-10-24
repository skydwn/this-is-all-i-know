# Stack Overflow and Shellcode Execution
Neste phrack vou demonstrar um buffer overflow stack-based, os primeiros passos para quem está interessado em
vulnerabilidades de corrupção de memória. O binário que vou utilizar para a demonstração é intencionalmente
vulnerável e é conhecido como Brainpan. Veja abaixo a depuração inicial do binário no Immunity Debugger:<br><br>
<img src="https://camo.githubusercontent.com/3fdf7e4740d0e700bf9abe6506ce2e8fb7a7ff6d6bce12976696eb5cbd68d40c/68747470733a2f2f692e696d6775722e636f6d2f3078556a35665a2e706e67"><br><br>

As instruções ao lado esquerdo são chamadas de opcodes, e cada uma delas representa uma operação e seus operandos.
Essas instrução ocupam a section .text no layout de memória do processo. Observe também abaixo, o memory layout
de um processo ativo.<br><br>

<img src="https://miro.medium.com/v2/resize:fit:640/format:webp/1*b3l-cBj4C1tII5Qs0mEvrw.png"><br><br>

Observe que entre a estrutura de memória e a section .text (onde estão as instruções), existem duas sections<br>
sendo elas .data, onde serão armazenadas as variáveis e .bss (buffer) onde será armazenado as alocações de memória<br><br>
Note que se os bytes (no caso de um overflow) transbordarem da section .bss, acaba sobrescrevendo os bytes na<br>
estrutura de memória, sendo ela a stack. Tudo que é preciso, é descobrir com quantos bytes o buffer é preenchido<br>
até transbordar para a stack.<br><br>
Para isso utilizei o pattern, ferramenta que gera uma string com padrões de 8 bytes (Ex: AAAAAAAABBBBBBBB) para determinar
qual padrão sobrescreve o primeiro endereço da stack, por exemplo, se JJJJJJJJ sobrescreveu o primeiro endereço da stack
quer dizer que com 72 bytes atingimos a stack. (Sendo que cada caractere ocupa 8 bytes).<br><br>

<img src="https://camo.githubusercontent.com/4d0045ce1abd40fc19d521678b49554ee96221a88cf3afda5ffd053f6ccc0f29/68747470733a2f2f692e696d6775722e636f6d2f4e4746693653582e706e67">
<hr>
<img src="https://camo.githubusercontent.com/85a620e628387a3b5d23354b56a6be7f2b1777cc40c1f2d9cbccb5f1a031cb6f/68747470733a2f2f692e696d6775722e636f6d2f6f77466c4b316a2e706e67"><br><br>

Observe que durante o envio do pattern (Primeira imagem), fazendo a depuração do binário (Segunda imagem) é possível
observar que a stack (Registrador ESP) foi completamente preenchida com o pattern gerado, portanto podemos utilizar o pattern_offset.rb para calcular quantos bytes foram utilizados até chegar ao 0x35724134 (\x34\x41\x72\x35)<br><br>

<img src="https://camo.githubusercontent.com/3feb47a972296602c552c2885df4ba069b34ccea279d8c3350d871b12a0e54de/68747470733a2f2f692e696d6775722e636f6d2f68466a753732672e706e67"><br><br>

Segundo o calculo feito pelo pattern_offset.rb, o offset do endereço 0x35724134 é de 524 bytes. Para confirmar<br>
essa informação, vou fazer uma prova de conceito utilizando um payload simples em perl, onde vou sobrescrever o EIP com meu nickname "kosu". Segue abaixo o payload:<br><br>
perl -e 'print "a"x524 . "kosu"' | ncat 192.168.15.14 9999<br><br>
<img src="https://camo.githubusercontent.com/a3f5a1aaa963c78fd768428172be717799e6b10284496f196be7f370a620ec88/68747470733a2f2f692e696d6775722e636f6d2f694175647839612e706e67"><br><br>

Agora basta converter os bytes no registrador EIP (0x75736f6b / \x6b\x6f\x73\x75) para texto plano e confirmamos
que o nosso payload deu certo, e que "kosu" preencheu perfeitamente o registrador EIP<br><br>
0x75736f6b = "usok" (na arquitetura do processador, os endereços de memória são organizados de trás pra frente, isso é conhecido como little-endian, por isso os bytes ficam ao contrário).<br><br>

Agora que obtemos o controle do EIP, vamos utilizar algum driver ou dependência desse binário e procurar um gadget (instrução/opcode) que podemos utilizar para fazer uma ROP Chain (encadear várias instruções e realizar uma ação maliciosa) ou então "pular" para a stack, onde vamos alocar o nosso shellcode. Segue abaixo a lista de DLLs utilizadas pelo binário e o gadget "JMP ESP" que eu encontrei na KERNELBASE.dll<br><br>

<img src="https://camo.githubusercontent.com/a366da9f5d1a046396c45cad7df0983315eee5b8fe59194db8f666af6e7c4502/68747470733a2f2f692e696d6775722e636f6d2f763945424f4b642e706e67">
<hr>
<img src="https://camo.githubusercontent.com/04bbbf8ced19603e8517d615f2a557cde00bf4c24695fcb4ecedc6cd935a4d8c/68747470733a2f2f692e696d6775722e636f6d2f4a4d3875664a732e706e67">

Gadget: JMP ESP - 0x311712f3 (\xf3\x12\x17\x31)<br>
Para melhorar a depuração do exploit, e para garantir que o gadget JMP ESP será direcionado ao registrador EIP, eu configurei um breakpoint no Immunity Debugger no gadget JMP ESP, e no momento que o endereço 0x311712f3 atingisse o registrador EIP a execução do processo pausaria. Segue abaixo o payload utilizado e o breakpoint atingido durante a depuração do exploit.<br><br>

perl -e 'print "a"x524 . "\xf3\x12\x17\x31"' | ncat 192.168.15.14 9999<br><br>
<img src="https://camo.githubusercontent.com/fafe94c3f118e27998713935403e626fa71b0920cab4746274f3eab437b0cbdf/68747470733a2f2f692e696d6775722e636f6d2f703749694a35362e706e67">
<hr>
<img src="https://camo.githubusercontent.com/3ab2074e18b60896fc497de8a799b581607e2f524bbc194802de9dd6ec606877/68747470733a2f2f692e696d6775722e636f6d2f4e4777795330642e706e67"><br><br>

Você pode notar que ao enviar o payload na primeira imagem, durante a depuração do código o breakpoint foi atingido e o endereço de JMP ESP chegou ao registrador EIP, estabilizando o payload para o próximo passo, que seria alocar o shellcode na stack (Registrador ESP), e após a execução do gadget JMP ESP o nosso shellcode será executado. Segue abaixo a geração do shellcode utilizando msfvenom e a execução do shellcode na stack<br><br>

<img src="https://camo.githubusercontent.com/91dbd2f58109788de51b53453f59254c317718f23d03461c86f612f9018323a5/68747470733a2f2f692e696d6775722e636f6d2f4d574642504c652e706e67">
<hr>
<img src="https://camo.githubusercontent.com/b8e4774d8b57688f5d087a84cabac349672e6a850fabafed2d641c05b90bcdae/68747470733a2f2f692e696d6775722e636f6d2f353034625450332e706e67"><br><br>

by sky / kosu<br>
Escrito em 26/09/2020
