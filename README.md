# ESP32 Assembly Language Unofficial Documentation (My Notes)

I haven't found any resource where knowledge about ESP32 assembly is collected so i created one.

All the knowledge I present here has been gained by debugging and self-testing, reading documentation, and testing scraps of information from the web.

There are some things I can be wrong about, so take this with a grain of salt. When you find a mistake or inaccuracy, don't hesitate to let me know.

**All project files created and tested on <mark>ESP-IDF</mark> using Visual Studio Code with esp-idf addon. Should work on plain device as well (idk).**

#### WARNING! Documentation still in progress!

# ESP32 pure asm (no esp-idf)

[assembly - Programming esp32 and esp8266 - Stack Overflow](https://stackoverflow.com/questions/63900236/programming-esp32-and-esp8266)

# Visual Studio Code - ESP-IDF addon

Jak dodajemy plik to musimy go rowniez zarejestrowac w CMakeLists.txt (w tym samym miejscu co sa te pliki)
idf_component_register(SRCS "main.c" "onewiredriver.S" INCLUDE_DIRS ".")

# ESP32 Assembly

Handbook i cala dokumentacja:

- **<mark>[DOC1-page] </mark>** http://loboris.eu/ESP32/Xtensa_lx%20Overview%20handbook.pdf
  
- **<mark>[DOC2-page]</mark>** https://dl.espressif.com/github_assets/espressif/xtensa-isa-doc/releases/download/latest/Xtensa.pdf
  
- [Demystifying Xtensa ISA](https://sachin0x18.github.io/posts/demystifying-xtensa-isa/) - Explained WindowRegisters
  
- [Xtensa Assembly Walkthrough](https://medium.com/@bayotosho/xtensa-lx7-assembly-walkthrough-by-example-c43f529bdeb1)
  

## W asm **0x0** (hex) = **0** (dec), nie rozroznia #0 i 0:

- 0x0 jako **adres** `mov a4, 0x0`

- 0x0 jako **stala** `movi a4, 0x0`

## Instrukcje w Xtensa ASM:

- `s32i a15, a14, 0` -> save **a15** to location **(a14 + 0)**
  
- `addi a15, a14, 1` -> add **1** to **a14** and save to **a15**
  
- `sub a13, a14, a15` -> **a13 = a14 - a15**
  
- `l32r a15, LED_PIN`-> load **literal** LED_PIN (constant value) into **a15**. Jeżeli stała (constant value) nie mieści się na **12-bitach**, tzn. nie możemy użyć **movi** to używamy **l32r** [What does a dangerous relocation error mean? - Stack Overflow](https://stackoverflow.com/questions/19532826/what-does-a-dangerous-relocation-error-mean)
  

## Budowa rejestrów (w przypadku kiedy wywołujemy inną funkcję z asm)

| Register | Use |
| --- | --- |
| a0  | Return address |
| a1 (sp) | Stack pointer |
| a2-a7 | Incoming arguments |
| a12-a15 | Calee-saved <br/>(do wykorzystania na potrzeby funkcji) |

~~W przypadku gdy **odbieramy w asm** to **a2-a7** to będą dane, które zwracamy (return), a **a12-a15**, argumenty funkcji (te, które otrzymaliśmy)~~

## Funkcje w ASM 1 - przekazywanie/zwracanie danych - informacje ogólne

Przykładowa funkcja w asm i odwołanie z C:

- [Example function in ASM + C/C++](https://github.com/technosf/XtensaAsmPOC)

### **Przekazywanie do funkcji:**

Pierwsze 6 argumentów jest przekazywane przez rejestry **(od a2 do a7)**, a reszta argumentów na stosie.

~~Z moich obserwacji wynika, że pierwsze dwa argumenty to: **a7, a3~~**

### **Zwracane z funkcji**

Zwracane są w rejestrach **(od a2 do a5)**, gdy trzeba więcej to ten co wywołuje przekazuje wskaźnik do miejsca, które potem wywołana funkcja zapełnia.

Z moich obserwacji wynika, że standardowe zwracanie odbywa się przez **a2**

## Funkcje w ASM 2 - wywoływane z C/C++

### **Każda funkcja, która chcemy wywołać z C/C++ zawiera**

- `.global nazwa_funkcji` pozwala na `extern` z C/C++
  
- `.align 4` <mark>TODO</mark>
  
  Dokładne wyjaśnienie tutaj: [what is the meaning of align - Stack Overflow](https://stackoverflow.com/questions/11277652/what-is-the-meaning-of-align-an-the-start-of-a-section)
  
- `entry a1, 32` ma za zadanie allokować **SP** dla funkcji oraz przesunąć okno rejestru o zadaną wartość **<mark>[DOC2-7]</mark>**
  
- `retw` wraca i dekrementuje rejestr WindowBase **<mark>[DOC2-6]</mark>**
  

**UWAGA!** Jeżeli funkcja zawiera `entry a1, 32` to nie można jej wywołać z innej funckji z asemblera!

### Przykładowe funkcje

1. `extern void mwait(void);` **<- w C/C++ (returns void, void args)**
  
  ```
  .text
  
  .global mwait
  .align 4
  mwait:
      entry a1, 32
      retw
  ```
  
2. `extern uint32_t mwait(uint32_t, uint32_t);` **<- w C/C++(returns uint, uint args)**
  
  Zwracamy dane do a2 **<mark>[DOC1-99]</mark>**.
  
  ```
  .text
  
  .global mwait
  .align 4
  mwait:
      entry a1, 32
  
                          // args: pierwszy argument w a2, drugi w a3 ...
      movi a2, 100        // returns: zwracamy w a2
  
      retw
  ```
  

### How to call it from C/C++

```
#include <stdio.h>

extern uint32_t mwait(uint32_t, uint32_t);

void app_main() {
    uint32_t x = mwait(12, 33);
}
```

## Funkcje w ASM 3 - wywoływane z ASM

**UWAGA!** Jeżeli wywołujemy funkcję (z ASM), która jest w ASM to ta **NIE** może zawierać `entry a1, 32`.

### Przykład:

**Caller (ta, która wywołuje):**

- `.global, .align, entry` **so we can call it from C/C++**
  

```
.global caller
.align 4
caller:
    entry a1, 32
    
    call4 callee        // for now the only option i found
                        // return address in a4
    retw
```

**Callee (wywoływana):**

```
callee:
    movi a15, 420
    jx a4                // for now the only return option i found
                         // (call4 -> return address in a4)
```

**C/C++ code for trying out:**

```
#include <stdio.h>

extern void caller(void);

void app_main() {
    caller();
}
```

**UWAGA!** I had tried to run it with `calln` and `ret` (ret returns to address in a0), but it didn't work, only `call4` and `jx a4` seems to work for me.

## Sterowanie GPIO

GPIO (general purpose i/o) :)

Useful links:

- [l32r example](https://esp32.com/viewtopic.php?t=8826&start=10)
  
- [movi przykład](https://github.com/darthcloud/esp32_highint5_gpio/blob/master/main/highint5.S)
  

### Stałe warte wspomnienia:

.literal używamy idk jak więc **<mark>TODO</mark>**

```
.literal GPIO_STATUS_W1TC_REG, 0x3FF4404C     # IDK wiec todo
.literal GPIO_OUT_W1TS_REG, 0x3FF44008        # set PIN (high)  [REGISTER addr]
.literal GPIO_OUT_W1TC_REG, 0x3FF4400C        # clear PIN (low) [REGISTER addr]
.literal GPIO_NUM_2, (1<<2)                   # GPIO 2
```

### Jak ustawiamy stany?

Stała (constant value) (0x3FF44008) nie mieści się na 12-bitach, tzn. nie możemy użyć **movi**, używamy **l32r**: [What does a dangerous relocation error mean? - Stack Overflow](https://stackoverflow.com/questions/19532826/what-does-a-dangerous-relocation-error-mean)

W taki sposób ustawiamy **LOW** na pinie:

```
.text
.literal GPIO_OUT_W1TC_REG, 0x3FF4400C        # clear PIN (low) [REGISTER addr]
.literal GPIO_NUM_2, (1<<2)                   # GPIO 2

.global mwait
.align 4
mwait:
    entry a1, 32

    l32r a14, GPIO_OUT_W1TC_REG
    l32r a15, GPIO_NUM_2
    s32i a15, a14, 0                          # save GPIO_NUM_2 to GPIO_OUT_W1TC_REG

    retw
```

# WS2812B Driver

Useful links:

- https://cdn-shop.adafruit.com/datasheets/WS2812B.pdf - Documentation
  
- [WS2812 Tutorial: Protocol for the WS2812B Programmable LED | Arrow.com](https://www.arrow.com/en/research-and-events/articles/protocol-for-the-ws2812b-programmable-led)
  
- [NeoPixels Revealed: How to (not need to) generate precisely timed signals | josh.com](https://wp.josh.com/2014/05/13/ws2812-neopixels-are-not-so-finicky-once-you-get-to-know-them/)
  

### Timing table

| Func | Description | Time | ±   |
| --- | --- | --- | --- |
| T0H | 0 code, high voltage time | 0.4 us | ±150 ns |
| T0L | 0 code, low voltage time | 0.85 us | ±150 ns |
| T1H | 1 code, high voltage time | 0.8 us | ±150 ns |
| T1L | 1 code, low voltage time | 0.45 us | ±150 ns |
| RESET | low voltage time | > 50 us |     |

**In words:**

- 0 = 0.4 us high, 0.85 us low
  
- 1 = 0.8 us high, 0.45 us low
  

### LED Control

To control one diode we need to send **24-bits**.

| G7... | ...G0 | R7... | ..R0 | B7... | ...B0 |
| --- | --- | --- | --- | --- | --- |

- 8-bit GREEN
  
- 8-bit RED
  
- 8-bit BLUE
  

**First, G7 is populated (HIGH bit is sent at first).**

Sending `10000000 00000000 00000000` places 1 to G7 and every other is 0.

For example sending `11111111 00000000 00000000` turns the LED green.
