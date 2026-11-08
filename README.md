# Laboratorio 04: SoC Zynq-7000

- **Asignatura:** Electrónica Digital II
- **Semestre:** 2026-2

## Integrantes del equipo

- **Samuel Hincapie Perilla**
- **Eduardo Felipe Camacho Lara**
- **Gabriel Alberto Rodríguez Rincón**

---

## Contenido

1. [Objetivo](#1-objetivo)
2. [Arquitectura del Zynq-7000](#2-arquitectura-del-zynq-7000)
3. [Diseño del SoC en Vivado](#3-diseño-del-soc-en-vivado)
4. [Desarrollo del software en Vitis](#4-desarrollo-del-software-en-vitis)
5. [Funcionamiento en la FPGA](#5-funcionamiento-en-la-fpga)
6. [Conclusiones](#6-conclusiones)
7. [Referencias](#7-referencias)

---

## 1. Objetivo

Implementar un **System on Chip (SoC)** sobre la Zybo Z7-10 usando el *Block Design* de Vivado, integrando el procesador ARM del Zynq-7000 con memoria y periféricos implementados en la lógica programable, y ejecutar sobre él una aplicación en C desarrollada en Vitis.

El flujo completo de la práctica es:

```
Diseño de hardware (Vivado) → Bitstream → Exportación (.xsa) → Plataforma y aplicación (Vitis) → Ejecución en la FPGA
```

---

## 2. Arquitectura del Zynq-7000

El Zynq-7000 integra en un mismo chip dos partes que trabajan juntas:

| Parte | Contenido | Cómo se programa |
|---|---|---|
| **PS** (*Processing System*) | Procesador ARM Cortex-A9, cachés L1 y L2, controlador de memoria DDR, periféricos (UART, USB, Ethernet, SPI, I2C, CAN, GPIO), temporizadores, DMA y JTAG | Software (C en Vitis) |
| **PL** (*Programmable Logic*) | Lógica reconfigurable Serie 7: LUTs, flip-flops, bloques DSP, BRAM y ruteo programable | Hardware (Block Design o HDL en Vivado) |

Las dos partes se comunican mediante buses **AXI**. En esta práctica, el procesador (PS) actúa como **maestro**: lee y escribe en periféricos implementados en la PL (memoria BRAM y GPIO) como si fueran posiciones de memoria.

![Arquitectura del Zynq-7000](imgs/zynq-7000.png)

---

## 3. Diseño del SoC en Vivado

### 3.1 Instalación de los archivos de la tarjeta

Se instalaron los *board files* de Digilent siguiendo la guía oficial, para que Vivado reconozca la **Zybo Z7-10** como placa. Esto permite que Vivado conozca la configuración del PS (memoria DDR, relojes, pines MIO) y los periféricos de la tarjeta (LEDs y botones), y los configure automáticamente.

### 3.2 Creación del proyecto

Se creó un proyecto RTL seleccionando la placa **Zybo Z7-10** en la pestaña `Boards`, en lugar de elegir solo el chip.

![Selección de la placa](imgs/01_proyecto_board.png)

### 3.3 Bloques del sistema

En `IP INTEGRATOR → Create Block Design` se agregaron los siguientes bloques:

| Bloque | Función en el sistema |
|---|---|
| **ZYNQ7 Processing System** | Representa el PS: el procesador ARM y sus periféricos. Entrega a la PL el reloj (`FCLK_CLK0`), la señal de reset (`FCLK_RESET0_N`) y el puerto maestro AXI (`M_AXI_GP0`) por el que el procesador accede a los periféricos de la PL |
| **Processor System Reset** | Toma el reset del PS y lo sincroniza con el reloj. Genera los resets para el interconector (`interconnect_aresetn`) y para los periféricos (`peripheral_aresetn`), de modo que todos salgan de reset de forma ordenada |
| **AXI Interconnect** | Conecta un maestro (el procesador) con varios esclavos (controlador de BRAM y GPIO). Según la dirección de cada acceso, dirige la transacción al periférico correspondiente |
| **AXI BRAM Controller** | Traduce las transacciones AXI del procesador en las señales de dirección, dato y habilitación que necesita la memoria BRAM |
| **Block Memory Generator** | La memoria BRAM implementada en la PL |
| **AXI GPIO** | Periférico de entradas y salidas de propósito general, con un canal para los 4 LEDs y otro para los 4 botones de la tarjeta |

![Bloques agregados](imgs/02_bloques_sueltos.png)

### 3.4 Conexiones

Se conectaron los bloques de memoria y reset:

![Bloques conectados](imgs/03_bloques_unidos.png)

Luego se agregaron desde la pestaña `Board` los GPIO **4 Buttons** y **4 LEDs**, y se conectaron sus señales de reloj, reset y AXI:

![GPIO de LEDs y botones](imgs/04_gpio.png)

Con `Regenerate Layout`, el diagrama final queda así:

![Diagrama de bloques final](imgs/05_diagrama_final.png)

En el diagrama se distinguen tres tipos de conexiones:

| Conexión | Origen | Destino |
|---|---|---|
| **Reloj** | `FCLK_CLK0` del PS | Todos los bloques de la PL y el puerto `M_AXI_GP0_ACLK` del PS |
| **Reset** | `FCLK_RESET0_N` → Processor System Reset | Interconector y periféricos |
| **Datos (AXI)** | `M_AXI_GP0` del PS → AXI Interconnect | AXI BRAM Controller y AXI GPIO |

### 3.5 Mapa de direcciones

En el `Address Editor` se asignaron las direcciones de los periféricos. Desde el procesador, cada periférico de la PL ocupa un rango del mapa de memoria; el puerto `M_AXI_GP0` del Zynq accede a la región que va de `0x4000_0000` a `0x7FFF_FFFF`.

El rango del AXI GPIO se redujo de 64K a **4K**. Sus registros (datos y dirección de cada canal, e interrupciones) ocupan muy pocos bytes, así que 4 KB son suficientes y se evita reservar espacio de direcciones que nunca se usará.

![Address Editor](imgs/06_address_editor.png)

| Periférico | Dirección base | Rango |
|---|---|---|
| `axi_bram_ctrl_0` | _[completar con la captura]_ | _[ ]_ |
| `axi_gpio_0` | _[completar con la captura]_ | 4K |

Estas direcciones quedan registradas en el archivo `xparameters.h` que genera Vitis, y son las que usa el programa en C para acceder a los LEDs y botones.

### 3.6 HDL Wrapper

El Block Design no se puede sintetizar directamente: Vivado necesita un módulo HDL de nivel superior. Con `Create HDL Wrapper` → `Let Vivado manage wrapper and auto-update` se generó un archivo que instancia el diseño de bloques y expone sus puertos externos. Con la opción automática, el wrapper se actualiza solo si se modifica el diagrama.

![HDL Wrapper](imgs/07_wrapper.png)

### 3.7 Run Block Automation

`Run Block Automation` aplica al bloque ZYNQ7 Processing System la configuración predefinida de la Zybo Z7-10 (memoria DDR, pines MIO, relojes) y crea los puertos externos `DDR` y `FIXED_IO`, que conectan el PS con la memoria y los pines fijos de la tarjeta.

![Run Block Automation](imgs/08_run_block_automation.png)

### 3.8 Bitstream y exportación del hardware

Se generó el bitstream y se exportó el hardware con `File → Export → Export Hardware`, **incluyendo el bitstream**. El resultado es el archivo **`.xsa`**, que contiene:

- La configuración del PS.
- El mapa de direcciones de los periféricos.
- El bitstream que programa la PL.

Este archivo es el puente entre Vivado y Vitis: con él, Vitis sabe qué hardware existe y en qué direcciones está.

![Bitstream generado](imgs/09_bitstream.png)

![Exportación del hardware](imgs/10_export_hardware.png)

---

## 4. Desarrollo del software en Vitis

### 4.1 Plataforma

Desde Vivado (`Tools → Launch Vitis IDE`) se abrió Vitis y se creó un **Platform Component** a partir del archivo `.xsa`, con el flujo `Hardware Design`. La plataforma se configuró con sistema operativo **standalone** (sin sistema operativo) sobre el procesador **ps7_cortexa9_0**.

![Creación de la plataforma](imgs/11_vitis_platform.png)

La plataforma reúne la descripción del hardware y los controladores (*drivers*) de los periféricos, como el del AXI GPIO, que la aplicación usa para acceder a ellos.

### 4.2 Aplicación

Sobre la plataforma se creó una **Application Component** vacía, usando el dominio creado por defecto, y se agregó el archivo `main.c` de la guía.

![Creación de la aplicación](imgs/12_vitis_application.png)

### 4.3 Compilación y ejecución

Con `Build` se compiló la aplicación sin errores. Con la Zybo conectada, `Run` programó la PL con el bitstream, cargó el programa en el procesador y lo ejecutó.

![Build y Run](imgs/13_build_run.png)

---

## 5. Funcionamiento en la FPGA

**Video:** [Funcionamiento del SoC en la Zybo Z7](_[https://drive.google.com/file/d/1iDhMxpr6cz7GVlGyuB_5qeMGZueUY2hL/view?usp=sharing]_)

https://drive.google.com/file/d/1iDhMxpr6cz7GVlGyuB_5qeMGZueUY2hL/view?usp=sharing
Se observó el comportamiento esperado:

- Los **LEDs parpadean** de forma continua.
- Al Activar un switch, el LED correspondiente permanece encendido mientras el switch siga en alto.

Esto confirma que el sistema completo funciona:

| Elemento | Evidencia |
|---|---|
| Procesador ARM ejecutando el programa | El patrón de parpadeo lo genera el código en C |
| Bus AXI y mapa de direcciones | El procesador escribe en los LEDs y lee los botones a través del AXI GPIO |
| Lógica programable | El AXI GPIO, el interconector y el controlador de BRAM están implementados en la PL con el bitstream exportado |

---

## 6. Conclusiones

---

## 7. Referencias

- FPGAs for Beginners. (2023, 28 de julio). *Zynq Part 1: Vivado block diagram (no Verilog/VHDL necessary!)* [Video]. YouTube. https://www.youtube.com/watch?v=UZ3FnZNlcWk
- AMD. (s. f.). *SoC AMD Zynq 7000*. https://www.amd.com/es/products/adaptive-socs-and-fpgas/soc/zynq-7000.html
- Digilent. (s. f.). *Installing Vivado Board Files for Digilent Boards*. https://digilent.com/reference/programmable-logic/guides/install-board-files
