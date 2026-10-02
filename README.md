# Projeto da ULA – Primeira Unidade

Unidade Lógica e Aritmética de 5 bits (sinal-magnitude) com decodificador BCD para displays de 7 segmentos, feita **somente com portas lógicas** no Quartus Prime Lite 21.1 e pronta para a placa **DE2-115** (FPGA Cyclone IV E **EP4CE115F29C7**).

Projeto da primeira unidade de **Sistemas Digitais (CIN0007)** – Engenharia da Computação, CIn-UFPE. 

## Como abrir o projeto

1. Baixe o repositório (**Code → Download ZIP**) e **extraia** o ZIP numa pasta sem acentos e sem espaços no caminho.
2. Dê duplo clique em **`SDprojeto.qpf`**.
3. Confira em **Assignments → Device** se o dispositivo é **Cyclone IV E – EP4CE115F29C7**.
4. Abra o `circuito_completo.bdf` (top-level) e compile com **Ctrl+L**.

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

Dentro da ULA, todas as operações são calculadas **em paralelo** e um multiplexador, controlado por `S`, escolhe qual resultado sai em `F` e no LED de status.

---

## Módulos

| Módulo | Arquivos | Função |
|--------|----------|--------|
| Somador/Subtrator | `soma`, `somador` | Soma e subtração em sinal-magnitude |
| Complemento a 2 | `complementa2` | Complemento a 2 de B |
| Comparadores | `comparadores_ULA`, `comparador_sinal`, `comp_bit_cascata` | Igual, maior e menor (sinal + magnitude em cascata) |
| Blocos lógicos | `blocos_logicos` | AND e XOR bit a bit |
| Multiplexadores | `MUX_Operacoes`, `MUX_Intermediario_operacoes_6bits`, `MUX_3p1` | Escolhem o resultado `F` e o status conforme `S` |
| Conversor | `Conversor` | Binário de 5 bits para BCD (dezena e unidade) |
| Decodificador | `Decodificador`, `Decoder_7seg`, `segmento_a` … `segmento_g` | BCD para display de 7 segmentos (ânodo comum) |
| Top-level | `circuito_completo` | Integra todos os módulos |

---

## Detalhamento dos módulos

### Somador/Subtrator

Faz soma e subtração entre dois números de 5 bits em sinal-magnitude. Internamente, **toda subtração vira uma soma**, usando complemento a 2, então não há subtrator dedicado.

- **Entradas:** `A[3:0]`, `SA`, `B[3:0]`, `SB` e `SELECT` (0 = soma, 1 = subtração).
- **Saídas:** `F[4:0]` (magnitude) e `SF` (sinal).

O circuito tem quatro partes:

1. **Controle (`SELECT`):** define se o operando B entra como está ou complementado. Combinando `SELECT` com o sinal de B:

   | SELECT / SB | 0 (+B) | 1 (−B) |
   |-------------|--------|--------|
   | 0 (soma) | +B | −B |
   | 1 (subtração) | −B | +B |

2. **Blocos de complemento a 2 (`complementa2`):** complementam um dos operandos quando necessário, em duas etapas: inverter todos os bits (NOT) e somar 1. Assim:
   - A − B = A + (complemento de 2 de B)
   - −A + B = (complemento de 2 de A) + B
   - −A − B = (complemento de 2 de A) + (complemento de 2 de B)
3. **Somadores completos (*full adders*):** ligados em cascata, formam um somador de 4 bits com propagação de carry. O carry do último estágio entra no cálculo do sinal do resultado.
4. **Lógica auxiliar (XOR, AND, OR):** as XOR decidem se os bits de B são invertidos (com base em `SELECT` e `SB`). As AND/OR tratam magnitude zero, evitando o **−0** no resultado.

### Complemento a 2 de B

É a operação `S = 010`. O bloco `complementa2` inverte os bits de B e soma 1, e a saída segue para o multiplexador. O mesmo bloco é usado dentro do somador/subtrator.

### Comparador

O comparador segue a lógica de **sinal e magnitude em cascata**. Ele tem três saídas exclusivas: `fio_maior` (A > B), `fio_igual` (A = B) e `fio_menor` (A < B).

**1. Comparador de sinal (`comparador_sinal`)**
Compara um único par de bits. Tabela verdade e equações:

| A | B | maior | igual | menor |
|---|---|-------|-------|-------|
| 0 | 0 | 0 | 1 | 0 |
| 0 | 1 | 0 | 0 | 1 |
| 1 | 0 | 1 | 0 | 0 |
| 1 | 1 | 0 | 1 | 0 |

- maior = A · B' (NOT + AND)
- igual = A ⊙ B (XNOR)
- menor = A' · B (NOT + AND)

É usado na comparação do **bit de sinal**. Se A é positivo e B é negativo, A é maior. No caso contrário, A é menor.

**2. Comparador de 1 bit com cascata (`comp_bit_cascata`)**
Estende o bloco anterior com as entradas `en_igual`, `en_maior` e `en_menor`, que trazem o resultado do estágio anterior. Seja `eq = A ⊙ B`:

- igual = en_igual · eq
- maior = en_maior + en_igual · A · B'
- menor = en_menor + en_igual · A' · B

Ou seja, se os bits mais significativos já desempataram, o resultado é apenas repassado. O bit atual só importa quando `en_igual = 1`.

**3. Comparador completo (`comparadores_ULA`)**
- Se os sinais são diferentes, o sinal decide: número positivo é maior que negativo.
- Se os sinais são iguais, a magnitude é comparada por quatro blocos `comp_bit_cascata` (`bit3` a `bit0`), do bit mais significativo para o menos.
- **Dois números negativos:** o resultado de maior/menor é **invertido**. Por exemplo, −8 < −7, embora 8 > 7.
- **Tratamento do zero:** como existem +0 e −0, uma porta NOR de 8 entradas detecta magnitude zero em A e B. Nesse caso `igual = 1`, e `maior` e `menor` ficam em 0.

### Blocos lógicos (AND/XOR)

- Cinco portas XOR e cinco portas AND, uma por bit (sinal + 4 de magnitude).
- Cada bit de A é combinado apenas com o bit de mesma posição em B.
- XOR dá 1 quando os bits são diferentes. AND dá 1 só quando os dois bits valem 1.
- **Saídas:** `xor0` a `xor4` e `and0` a `and4`, que vão para o multiplexador.
- Exemplo: A = 0101 e B = 0011 → XOR = 0110 e AND = 0001.

### Multiplexadores

O multiplexador é o componente central de seleção da ULA.

- **`MUX_Operacoes`:** recebe os resultados já calculados de soma, subtração, complemento a 2, AND e XOR e, guiado por `S[2:0]`, escolhe qual vai para `F`. Há um multiplexador 8×1 por bit de `F`, todos com o mesmo seletor. O sinal do resultado segue o mesmo formato das entradas (0 = positivo, 1 = negativo).
- **`MUX_Intermediario_operacoes_6bits`:** etapa intermediária de seleção dos bits de `F`.
- **`MUX_3p1`:** escolhe, entre `igual`, `maior` e `menor`, qual vai para o LED de status `compara`.
- Nas operações que retornam um booleano (`=`, `>`, `<`), o resultado **não passa** pelo multiplexador de dados. O status vai direto ao LED, e `F` fica em zero.

### Conversor binário para BCD

Transforma o resultado de 5 bits (0 a 30) em dois dígitos decimais.

- **Entrada:** `A[4:0]` (o bit 4 é o carry).
- **Saídas:** `S5, S4` (dezena, de 0 a 3) e `S3` a `S0` (unidade, de 0 a 9).
- O valor 31 não ocorre na ULA e é tratado como *don't care*.
- Exemplo: 23 → dezena 2 (`S5 S4 = 10`) e unidade 3 (`S3..S0 = 0011`).

### Decodificador de 7 segmentos

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

## Integração (`circuito_completo`)

No circuito completo, os módulos ficam ligados assim:

1. **Entradas:** `A`, `B` e `S` vêm das chaves da placa. Os bits de sinal e de magnitude de `A` e `B` também alimentam LEDs que replicam as entradas.
2. **Displays de A e B:** cada um tem seu próprio `Decodificador`, ligado aos bits `A3..A0` (ou `B3..B0`), com `A4` no terra e `Seletor` em nível 1. Assim os dois ficam sempre acesos.
3. **Operações em paralelo:** `A`, `B` e `S` entram ao mesmo tempo no somador/subtrator (com `SELECT` vindo de `S0`), no complemento a 2, no comparador e nos blocos AND/XOR.
4. **Seleção:** `MUX_Operacoes` escolhe o resultado que vai para `F`, e `MUX_3p1` escolhe o status do LED `compara`, ambos controlados por `S`.
5. **Saída F:** os 6 bits de `F` vão para os LEDs e, passando pelo conversor e pelo decodificador, para os displays de 7 segmentos. O bit de sinal de `F` vai direto ao LED.
6. **Controle do display de F:** o `Seletor` do decodificador de `F` vale 1 apenas quando `S2 = 0` e `S1 = 0`, ou seja, na soma e na subtração. Nas outras operações o display apaga.

---

## Exemplos de funcionamento

| A | B | S | Resultado |
|---|---|---|-----------|
| +7 | +8 | `000` (soma) | F = +15 |
| +9 | +9 | `000` (soma) | F = +18 (usa o carry) |
| +3 | +5 | `001` (subtração) | F = −2 (LED de sinal aceso) |
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
- O projeto é feito integralmente com portas lógicas, seguindo a metodologia de tabela verdade, mapa de Karnaugh e simplificação booleana, com testes por simulação (waveform).
