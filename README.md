# Projeto da ULA – Primeira Unidade

Unidade Lógica e Aritmética de 5 bits (sinal-magnitude) com decodificador BCD para displays de 7 segmentos, feita **somente com portas lógicas** no Quartus Prime Lite 21.1 e pronta para a placa **DE2-115** (FPGA Cyclone IV E **EP4CE115F29C7**).

Disciplina de Sistemas Digitais – Engenharia da Computação, CIn-UFPE.

---

## Como abrir o projeto

1. Baixe o repositório (**Code → Download ZIP**) e **extraia** o ZIP numa pasta sem acentos e sem espaços no caminho.
2. Dê duplo clique em **`SDprojeto.qpf`**.
3. Confira em **Assignments → Device** se o dispositivo é **Cyclone IV E – EP4CE115F29C7**.
5. Abra o `circuito_completo.bdf` (top-level) e compile com **Ctrl+L**.

---

## Operações

O seletor `S` (3 bits) escolhe a operação:

| S2 | S1 | S0 | Função |
|----|----|----|--------|
| 0 | 0 | 0 | F = A + B |
| 0 | 0 | 1 | F = A − B |
| 0 | 1 | 0 | Complemento a 2 de B |
| 0 | 1 | 1 | F = (A = B) |
| 1 | 0 | 0 | F = (A > B) |
| 1 | 0 | 1 | F = (A < B) |
| 1 | 1 | 0 | F = A AND B (bit a bit) |
| 1 | 1 | 1 | F = A XOR B (bit a bit) |

**Entradas**
- `A` e `B`: 5 bits cada (1 bit de sinal + 4 de magnitude). Sinal = 1 indica número negativo. O usuário não precisa se preocupar com complemento a 2 na entrada.
- `S`: 3 bits, seleção da operação.

**Saídas**
- `F`: 6 bits (5 de magnitude + 1 de sinal), em binário (não complementado a 2). Qualquer complementação necessária é feita internamente.
- `compara`: 1 LED de status para as operações `=`, `>` e `<`.
- 6 LEDs replicando `F`. Neles também aparecem AND, XOR, complemento a 2, soma e subtração.
- Displays de 7 segmentos para `A`, `B` e `F`.

> Os displays de `F` funcionam **apenas na soma e na subtração**. Nas demais operações ficam apagados.

---

## Visão geral em blocos

```
A (5 bits) ──┐
B (5 bits) ──┼──► [ ULA ] ──► F (6 bits) ──┬──► [ Decodificador ] ──► Displays de F
S (3 bits) ──┘       │                     └──► LEDs de F
                     └──► Status (1 LED)

A ──► [ Decodificador ] ──► Displays de A
B ──► [ Decodificador ] ──► Displays de B
```

Dentro da ULA, as operações são calculadas em paralelo e um multiplexador, controlado por `S`, escolhe qual resultado sai em `F` e no LED de status.

---

## Módulos

| Módulo | Arquivos | Função |
|--------|----------|--------|
| Somador | `soma`, `somador` | Soma e subtração em sinal-magnitude |
| Complemento a 2 | `complementa2` | Complemento a 2 de B |
| Comparadores | `comparadores_ULA`, `comparador_sinal`, `comp_bit_cascata` | Igual, maior e menor (sinal + magnitude em cascata) |
| Blocos lógicos | `blocos_logicos` | AND e XOR bit a bit |
| Multiplexadores | `MUX_Operacoes`, `MUX_Intermediario_operacoes_6bits`, `MUX_3p1` | Escolhem o resultado `F` e o status conforme `S` |
| Conversor | `Conversor` | Binário de 5 bits para BCD (dezena e unidade) |
| Decodificador | `Decodificador`, `Decoder_7seg`, `segmento_a` … `segmento_g` | BCD para display de 7 segmentos (ânodo comum) |
| Top-level | `circuito_completo` | Integra todos os módulos |

### Blocos lógicos (AND/XOR)

- Cinco portas XOR e cinco portas AND, uma por bit (sinal + 4 de magnitude).
- Cada bit de A é combinado apenas com o bit de mesma posição em B.
- XOR dá 1 quando os bits são diferentes. AND dá 1 só quando os dois bits valem 1.
- Exemplo: A = 0101 e B = 0011 → XOR = 0110 e AND = 0001.

### Comparador

O comparador segue a lógica de **sinal e magnitude em cascata**.

1. **Comparador de sinal:** um XNOR indica se os sinais são iguais. Se A é positivo e B é negativo, A é maior. No caso contrário, A é menor.
2. **Comparador de 1 bit com cascata:** quando os sinais são iguais, a magnitude é comparada bit a bit, do mais significativo para o menos. Cada bloco tem as entradas `en_igual`, `en_maior` e `en_menor`, que trazem a decisão do bloco anterior. Se um bit já desempatou, esse resultado é apenas repassado aos blocos seguintes.
3. **Dois números negativos:** o resultado de maior/menor é **invertido**. Por exemplo, −8 < −7, embora 8 > 7.
4. **Tratamento do zero:** como existem +0 e −0, uma porta NOR de 8 entradas detecta magnitude zero em A e B. Nesse caso `igual = 1`, e `maior` e `menor` ficam em 0.

### Decodificador de 7 segmentos

- O `Conversor` separa o valor em **dezena** (S5, S4) e **unidade** (S3 a S0). Exemplo: 23 vira dezena 2 e unidade 3.
- O `Decodificador` contém o conversor e dois blocos `Decoder_7seg`, um para a dezena e outro para a unidade.
- Cada `Decoder_7seg` tem sete módulos, um por segmento (a a g), feitos com portas NOT, AND e OR.
- Os displays da DE2-115 são de **ânodo comum**: o segmento acende com nível 0 e apaga com 1.
- O sinal `Seletor` apaga o display quando vale 0 (todas as saídas vão para 1). Com `Seletor = 1`, os segmentos mostram o dígito.
- Nos displays de `A` e `B`, o mesmo bloco é reaproveitado, com `A4` no terra (a magnitude só tem 4 bits) e `Seletor` fixo em 1. O bit de sinal não passa pelo decodificador e vai direto para o LED.

**Equações dos segmentos** (S' é o `Seletor` invertido; saída 1 = segmento apagado):

| Segmento | Equação |
|----------|---------|
| a | S' + A2·A1'·A0' + A3'·A2'·A1'·A0 |
| b | S' + A2·A1'·A0 + A2·A1·A0' |
| c | S' + A2'·A1·A0' |
| d | S' + A2'·A1'·A0 + A2·A1'·A0' + A2·A1·A0 |
| e | S' + A0 + A2·A1' |
| f | S' + A2'·A1 + A2·A1·A0 + A3'·A2'·A0 |
| g | S' + A3'·A2'·A1' + A2·A1·A0 |

Os dígitos de 10 a 15 não ocorrem, então são tratados como *don't care* nos mapas de Karnaugh.

---

## Exemplos de funcionamento

| A | B | S | Resultado |
|---|---|---|-----------|
| +5 | −6 | `100` (A > B) | `compara = 1` (os sinais já decidem) |
| −6 | +5 | `101` (A < B) | `compara = 1` |
| −7 | −8 | `100` (A > B) | `compara = 1` (negativos: a ordem se inverte) |
| +0 | −0 | `011` (A = B) | `compara = 1` |
| +0 | −0 | `100` ou `101` | `compara = 0` |
| 0101 | 0011 | `110` (AND) | `F = 00001` |
| 0101 | 0011 | `111` (XOR) | `F = 00110` |

---

## Observações

- O maior valor mostrado nos displays de `F` é 30 (15 + 15), o máximo possível em 6 bits. O valor 31 não ocorre na ULA.
- Os números A e B são limitados a 15 de magnitude, e o sinal é mostrado separadamente em LED.
- O projeto é feito integralmente com portas lógicas, sem `if`, `case` ou descrição comportamental em HDL.
