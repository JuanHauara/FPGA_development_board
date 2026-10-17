# Lattice FPGA development board

## Memoria RAM

La memoria prevista para el producto final es la **ISSI IS61WV5128BLL-10TLI**, por su menor precio en la comparación realizada. Para esta placa de desarrollo se utilizará la **Infineon CY7C1049G30-10ZSXI**: al seleccionar los componentes (2 de octubre de 2026), de estas dos referencias solo la Infineon estaba disponible en LCSC.

También se contempla la **ISSI IS61WV5128EDBLL-10TLI** como alternativa: incorpora **ECC interno que corrige un error de un bit por byte**, transparente para la FPGA.

Las tres referencias son **compatibles pin a pin** y pueden compartir el mismo footprint. Se montarán **dos chips de 512K × 8**, para un total de **1 MiB de RAM**, trabajando a **3,3 V**. En la ISSI con ECC, elegir el sufijo **`-10TLI`** para conservar el encapsulado y la velocidad indicados.

| Característica | ISSI elegida para el producto final | ISSI alternativa con ECC | Infineon para la placa de desarrollo |
|---|---|---|---|
| Modelo | IS61WV5128BLL-10TLI | IS61WV5128EDBLL-10TLI | CY7C1049G30-10ZSXI |
| Organización | 512K × 8 bits | 512K × 8 bits | 512K × 8 bits |
| Capacidad por chip | 512 KiB (4 Mbit) | 512 KiB (4 Mbit) | 512 KiB (4 Mbit) |
| Interfaz | SRAM asíncrona, direcciones y datos en paralelo | SRAM asíncrona, direcciones y datos en paralelo | SRAM asíncrona, direcciones y datos en paralelo |
| Encapsulado | TSOP-II, 44 pines, paso de 0,8 mm | TSOP-II, 44 pines, paso de 0,8 mm | TSOP-II, 44 pines, paso de 0,8 mm |
| Alimentación | 2,4–3,6 V | 2,4–3,6 V | 2,2–3,6 V |
| Tiempo de acceso | 10 ns | 10 ns | 10 ns |
| Temperatura de operación | −40 a +85 °C | −40 a +85 °C | −40 a +85 °C |
| ECC interno | No | Sí, transparente para la FPGA | Sí, transparente para la FPGA |

Para permitir el intercambio entre las tres referencias, el controlador debe contemplar hasta **5 ns para liberar el bus** después de desactivar `/OE` o `/CE`. La Infineon también requiere **100 µs de espera tras estabilizar la alimentación** antes de operar.

Referencias: [datasheet ISSI BLL](https://www.issi.com/WW/pdf/61-64WV5128Axx-Bxx.pdf), [datasheet ISSI EDBLL con ECC](https://www.issi.com/WW/pdf/61-64WV5128EDBLL.pdf), [datasheet Infineon](https://www.infineon.com/assets/row/public/documents/10/49/infineon-cy7c1049g-and-cy7c1049ge-are-high-performance-cmos-fast-static-ram-devices-with-embedded-ecc-datasheet-en.pdf) e [Infineon en LCSC — C2950234](https://www.lcsc.com/product-detail/C2950234.html).

### Selección de los dos bancos RAM (8086)

Referencia rápida para implementar el bus de **16 bits del 8086** en la FPGA. El IBM PC original usaba un **8088 con bus externo de 8 bits**; esta organización corresponde al 8086.

**Bus de datos: una mitad para cada chip.**

- **SRAM baja (direcciones pares):** `CPU D0–D7`.
- **SRAM alta (direcciones impares):** `CPU D8–D15`.

**Bus de direcciones: las mismas conexiones en ambos chips.**

```text
CPU          Ambas SRAM
A1    ────→  A0
A2    ────→  A1
...          ...
A19   ────→  A18
```

**CPU A0 selecciona el banco bajo; no se conecta a A0 de las SRAM.**

Durante un acceso al rango de RAM, los bancos se seleccionan así (`/BHE` es activa en 0):

| A0 | /BHE | Banco seleccionado |
|---:|---:|---|
| 0 | 0 | Ambos: palabra de 16 bits en dirección par |
| 0 | 1 | Bajo: byte en dirección par |
| 1 | 0 | Alto: byte en dirección impar |
| 1 | 1 | Ninguno |

- **Retener con ALE** la dirección completa `A0–A19` y `/BHE`; usar esos valores durante todo el ciclo.
- Los dos `/OE` pueden compartir la señal de **lectura de memoria**; los dos `/WE`, la de **escritura de memoria**. Mantenerlas inactivas fuera de la operación correspondiente.
- Una palabra en dirección **impar** requiere **dos ciclos**; una palabra en dirección par usa ambos bancos en un ciclo.

Referencia: [datasheet Intel 8086, selección A0 y /BHE (p. 2)](https://www.alldatasheetcn.com/html-pdf/1154707/INTEL/8086/349/2/8086.html) y [organización de memoria (p. 6)](https://www.alldatasheetcn.com/html-pdf/1154707/INTEL/8086/1061/6/8086.html).

## Clock de la CPU y ajuste de fase

### Generación del clock (función del 8284)

- **Frecuencia nominal tipo IBM PC:** `14,3181818… MHz ÷ 3 = 4,77272727… MHz`.
- **Divisor en HDL:** contador módulo 3 (`0 → 1 → 2 → 0`), salida **registrada**, alta un ciclo y baja dos. Evitar decodificación combinacional directa como clock.
- **Duty cycle:** 33,33 % alto; aproximadamente **69,84 ns alto / 139,68 ns bajo** antes del adaptador de nivel.

Ejemplo con oscilador de **25 MHz** y una PLL de la MachXO3LF:

```text
25 MHz → PLL (VCO = 787,5 MHz)
           ├─ CLKOP  = 262,5 MHz      → realimentación interna fija (INT_OP)
           ├─ CLKOS  = 14,3181818… MHz → contador ÷3 → adaptador → CLK de CPU
           └─ CLKOS2 = 14,3181818… MHz → lógica FPGA, con fase ajustable
```

**Divisores:** `CLKI_DIV=2`, `CLKFB_DIV=21`, `CLKOP_DIV=3`, `CLKOS_DIV=55`, `CLKOS2_DIV=55`. Con **50 MHz**, cambiar solo `CLKI_DIV=4`. **Propuesta calculada, pendiente de validar en Diamond** (PFD/VCO, divisores y temporización). La precisión depende del oscilador.

### Adaptación de 3,3 V a 5 V

- **SN74LVC4245APW:** `VCCA=5 V` (alimentación CPU), `VCCB=3,3 V`; `FPGA → B → A → CLK de CPU`, **`DIR=0`, `/OE=0` fijos**. No compartirlo con el bus bidireccional: los controles afectan a sus ocho canales.
- CLK del **80C86A** requiere un alto mínimo de `VCC−0,8 V` (**4,2 V a 5 V**); la FPGA de 3,3 V no lo garantiza.
- El retardo **desplaza el clock, no cambia su frecuencia**; no se compensa cambiando la frecuencia de la PLL. Puede modificar el duty cycle: `t_alto_salida = t_alto_entrada + tPHL − tPLH`.

### Ajustar solo la fase interna

El registro que genera el clock, la salida de la FPGA, el adaptador y las pistas retrasan el clock que llega a la CPU; las señales del bus también tardan en propagarse. **Ajustar la fase desplaza los flancos internos, sin cambiar la frecuencia**, para capturar datos cuando están estables y presentar señales a tiempo a la CPU. No es obligatorio si ya se cumplen los tiempos.

**Cómo elegir el ajuste:** partir de fase 0°, declarar los relojes y sus relaciones, incluir los retardos mínimos/máximos de CPU, adaptadores y pistas, y analizar **setup/hold** (datos estables antes/después del flanco) en Diamond. Considerar el clock que realmente llega al pin de CPU. Modificar solo la fase de CLKOS2 en la IP y repetir el análisis. Retrasar una captura puede dar más tiempo para que lleguen los datos, pero reduce el margen antes de su siguiente cambio; comprobar ambos límites, no solo un retardo típico.

- **Fijos: CLKOP y CLKOS. Ajustable: CLKOS2.** La realimentación no admite ajuste de fase. Mantener el contador de CPU en **CLKOS**; si usa CLKOS2, también se desplaza el clock de CPU.
- Configurar la fase en la IP. Para ajuste dinámico: `PHASESEL=01` selecciona CLKOS2; controlar `PHASEDIR`/`PHASESTEP` según la guía. Adelantar la fase puede generar pulsos espurios.
- **Ajustar con CPU en RESET** y liberar después de estabilizar el clock; sincronizar `LOCK` antes de usarlo en esa lógica.

Referencias: [guía sysCLOCK MachXO3, págs. 21, 27 y 29](https://www.latticesemi.com/view_document?document_id=50124), [datasheet SN74LVC4245A](https://www.ti.com/lit/ds/symlink/sn74lvc4245a.pdf) y [datasheet local Intel 80C86A, págs. 15–16 del PDF](Datasheet/IBM_PC/Intel%2080C86A%20CPU.pdf).

## Entradas de reloj de la FPGA (MachXO3LF)

### Qué pin elegir

| Necesidad | Pin recomendado | Función |
|---|---|---|
| Distribuir un reloj externo sin cambiar su frecuencia | `PCLKTx_y` | Entrada a la **red global de reloj**, que distribuye el clock a la lógica secuencial de toda la FPGA. No requiere PLL. |
| Generar otra frecuencia a partir de un reloj externo | `L_GPLLT_IN` o `R_GPLLT_IN` | Entrada dedicada de referencia a la **PLL izquierda o derecha**. La PLL puede multiplicar/dividir la frecuencia y alimentar la red global con el reloj resultante. |
| Realimentar externamente una PLL | `L_GPLL[T/C]_FB` o `R_GPLL[T/C]_FB` | **Entrada de feedback** para configuraciones que requieren realimentación externa; no es la entrada de referencia `_IN` ni una salida de reloj. Con realimentación interna no requiere conexión externa. |

Los pines `GPLL..._IN` son la ruta recomendada hacia la PLL, aunque existen otras rutas internas posibles. Las frecuencias de entrada y salida deben cumplir los límites de la PLL; configurar y comprobarlas en Diamond. [Guía sysCLOCK, §§4 y 11.3](https://www.latticesemi.com/view_document?document_id=50124).

### Cómo leer los nombres

- **`T` = True (+), `C` = Complement (−):** forman un par para reloj diferencial. Para un reloj de una sola señal (*single-ended*), elegir `PCLKT` o `GPLLT_IN`, según el uso. `PCLKC` no es una segunda entrada independiente de reloj single-ended.
- **`PCLK` = Primary Clock.** En `PCLKT4_0`, `4` indica el banco y `0` el índice de entrada de reloj. El reloj global puede llegar a toda la FPGA; no queda limitado a ese banco.
- **`L` / `R` = Left / Right:** identifican la PLL izquierda/derecha. **`IN`** es referencia y **`FB`** es realimentación.
- **Uso como GPIO:** si no se usa su función de reloj/PLL, estos pines pueden ser entradas o salidas normales, incluidos los `_FB`. En ese modo, `C` no invierte la señal ni obliga a usar su pareja `T`.

### Referencia rápida en J3

| Contacto de J3 | Bola de la FPGA | Función |
|---|---|---|
| — | — | **J3 no expone pines dedicados `PCLK` ni `GPLL`.** |

J3 pertenece al banco 0, pero sus entradas dedicadas de reloj no llegan a este conector. El reloj de **12 MHz de la placa** entra por **C8 (`PCLKT0_1`)**, fuera de J3. Para conectar un reloj externo a una entrada dedicada, usar J4, J6 o J8.

### Referencia rápida en J4

| Contacto de J4 | Bola de la FPGA | Función |
|---|---|---|
| 19 | H14 | `PCLKT1_0`: entrada de reloj global |
| 35 | D16 | `R_GPLLT_IN`: referencia externa de la PLL derecha |

Ambos pertenecen al **banco 1**. Para reloj diferencial, sus parejas `C` son, respectivamente, **J4-25** (H16, `PCLKC1_0`) y **J4-34** (E14, `R_GPLLC_IN`). La realimentación externa de la PLL derecha está en **J4-37** (D14, `R_GPLLT_FB`), con su pareja **J4-30** (E15, `R_GPLLC_FB`).

### Referencia rápida en J6

| Contacto de J6 | Bola de la FPGA | Función |
|---|---|---|
| 17 | T9 | `PCLKT2_1`: entrada de reloj global |
| 30 | T7 | `PCLKT2_0`: entrada de reloj global |

Ambos pertenecen al **banco 2**. Para reloj diferencial, sus parejas `C` son, respectivamente, **J6-19** (P9, `PCLKC2_1`) y **J6-28** (R8, `PCLKC2_0`).

### Referencia rápida en J8

| Contacto de J8 | Bola de la FPGA | Función |
|---|---|---|
| 7 | L2 | `PCLKT3_0`: entrada de reloj global |
| 14 | J1 | `PCLKT4_0`: entrada de reloj global |
| 28 | E1 | `PCLKT5_0`: entrada de reloj global |
| 27 | E2 | `L_GPLLT_IN`: referencia externa de la PLL izquierda |

Para reloj diferencial, las parejas `C` de esos cuatro pines son, respectivamente, **J8-6, J8-17, J8-25 y J8-37**. Usar niveles eléctricos compatibles con el banco y configurar el estándar de E/S correspondiente en Diamond.

Referencias: [Hardware Checklist, §13](https://www.latticesemi.com/view_document?document_id=50516), [datasheet MachXO3, §§2.4 y 4](https://www.latticesemi.com/view_document?document_id=50121) y [guía local del Starter Kit](MachXO3LF_starter_kit/FPGA-EB-02036-1-4-MachXO2-MachXO3-Starter-Kit-User-Guide.pdf), tablas 7.2–7.5 (páginas 18–21, columna **-6900C**) y esquemas del banco 0 y banco 1 (páginas 32–33).

## Bus ISA de 8 bits — IBM PC original (5150)

Conector de **62 contactos**, con **8 bits de datos y 20 de dirección**. **Cara A:** componentes de la tarjeta; **cara B:** soldaduras. **A1/B1** quedan junto al panel trasero. El prefijo `/` indica una señal **activa en 0**.

### Pinout

```text
       Cara B                  Cara A
Pin   Señal             Pin   Señal
B1    GND               A1    /I/O CH CK
B2    RESET DRV         A2    D7
B3    +5 V              A3    D6
B4    IRQ2              A4    D5
B5    -5 V              A5    D4
B6    DRQ2              A6    D3
B7    -12 V             A7    D2
B8    Reservado         A8    D1
B9    +12 V             A9    D0
B10   GND               A10   I/O CH RDY
B11   /MEMW             A11   AEN
B12   /MEMR             A12   A19
B13   /IOW              A13   A18
B14   /IOR              A14   A17
B15   /DACK3            A15   A16
B16   DRQ3              A16   A15
B17   /DACK1            A17   A14
B18   DRQ1              A18   A13
B19   /DACK0            A19   A12
B20   CLK               A20   A11
B21   IRQ7              A21   A10
B22   IRQ6              A22   A9
B23   IRQ5              A23   A8
B24   IRQ4              A24   A7
B25   IRQ3              A25   A6
B26   /DACK2            A26   A5
B27   T/C               A27   A4
B28   ALE               A28   A3
B29   +5 V              A29   A2
B30   OSC               A30   A1
B31   GND               A31   A0
```

### Función de las señales

- **A0–A19:** dirección de memoria o puerto de E/S; hasta **1 MiB** de memoria.
- **D0–D7:** bus bidireccional; transfiere un byte por acceso.
- **/MEMR:** lectura de memoria. **/MEMW:** escritura de memoria.
- **/IOR:** lectura de un puerto de E/S. **/IOW:** escritura de un puerto de E/S.
- **ALE:** mismo pin ALE de la CPU, sirve para identificar la fase de dirección de un ciclo de la CPU.
- **AEN:** en `1`, el controlador DMA tiene el control del bus.
- **I/O CH RDY:** normalmente `1`; la tarjeta lo baja a `0` para añadir esperas.
- **IRQ2–IRQ7:** la tarjeta solicita una interrupción mediante un flanco `0→1`.
- **DRQ1–DRQ3:** la tarjeta solicita DMA poniendo su línea en `1` hasta recibir `/DACK`.
- **/DACK1–/DACK3:** el controlador confirma y selecciona el canal DMA correspondiente.
- **/DACK0:** indica el ciclo de refresco de la RAM dinámica.
- **T/C:** pulso alto cuando un canal DMA alcanza su cuenta terminal.
- **/I/O CH CK:** la tarjeta comunica un error de paridad; puede provocar una NMI.
- **RESET DRV:** reinicia la lógica de las tarjetas cuando está en `1`.
- **CLK:** reloj del sistema, aproximadamente **4,77 MHz**, con un 33 % de ciclo alto.
- **OSC:** reloj de **14,31818 MHz**, con un 50 % de ciclo alto.
- **+5 V, -5 V, +12 V y -12 V:** alimentaciones; **GND:** masa común.

**B8 está reservado en el 5150**. Las señales lógicas son compatibles con TTL de 5 V.

Referencia: [IBM PC Technical Reference, agosto de 1981, págs. 2-8 a 2-12; pinout en 2-9](https://www.minuszerodegrees.net/manuals/IBM_5150_Technical_Reference_6025005_AUG81.pdf#page=28).
