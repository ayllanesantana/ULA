Explicações

SOMA: é um circuito simples que soma um bit de A com um bit de B, é usado dentro do Complementa 2 e do somador.

COMPLEMENTA 2: faz o complemento apenas quando o bit de sinal é negativo, nesse caso ( SA = 1 ) o XOR invertem as entradas e 1 entra no Cin.

SOMADOR/SUBTRATOR: o que diz se vai ser uma soma ou não é o SELECT, pq ele inverte o sinal de B.
A e B passa pelo Complementa2 ( que so complementa se for negativo ), e realiza a soma bit a bit pra depois descomplementar.
o Comp2 tem 6 bits de entrada, como A e B tem apenas 5bits eu coloco em A4 um bit a mais com valor 0 para que A e B passe a ter 6 bits.
