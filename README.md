# Adquisición de Datos Transitorios para Caracterización de Potencia RF

## Descripción General
Este repositorio contiene el firmware en C y las herramientas computacionales asociadas a un sistema de adquisición de datos (DAQ) de alta velocidad basado en la arquitectura STM32. El propósito central de esta instrumentación es la captura de la respuesta dinámica (respuesta al escalón) y el registro detallado del régimen transitorio en módulos atenuadores digitales. 

Esta caracterización experimental constituye una fase fundamental para el modelado matemático de plantas de radiofrecuencia. Al proveer datos empíricos de alta fidelidad, el sistema permite realizar una rigurosa **identificación paramétrica del sistema** físico. Este procedimiento es esencial para extraer la función de transferencia empírica, analizar la dinámica del hardware y construir modelos matemáticos precisos del comportamiento de la potencia en transmisores 5G mmWave operando en la banda de 25 GHz.
![alt text](ima2-1.png)
## Arquitectura de Hardware e Interfaz
El sistema orquesta múltiples periféricos operando a nivel de hardware para garantizar latencia determinista en la captura del transitorio:
*   **Microcontrolador:** Familia STM32L4 (Cortex-M4).
*   **Acondicionamiento RF:** Controladores de activación para detectores de potencia RMS LTC5596.
*   **Conmutación:** Control paralelo de 6 bits para atenuador digital (Resolución teórica de perturbación al escalón: de **16 dB** a **0 dB** en tiempo discreto).

## Arquitectura de Adquisición y Sincronización
La extracción de datos obvia la sobrecarga computacional de la CPU mediante una topología en cascada diseñada específicamente para no alterar la respuesta natural del sistema:
1.  **Base de Tiempo (TIM2):** Genera interrupciones físicas (TRGO) a una tasa estricta de **500 kHz** ($T_s$ = **2 µs**).
2.  **Conversión (ADC1):** Opera en *Scan Mode* para adquirir las tensiones correspondientes a la potencia de entrada y salida simultáneamente a una resolución de 12 bits.
3.  **Transporte de Memoria (DMA1):** Transfiere los arreglos cuantificados a los registros SRAM en una operación de disparo único hasta completar un bloque de 5000 muestras (**5 ms** de ventana de observación transitoria pura).
4.  **Transmisión (LPUART1):** Envía la trama binaria estructurada (Cabecera + Índice de Escalón + Arreglo de Datos) hacia la estación de trabajo a **460800 baudios**.

Para consultar la demostración de la estabilidad temporal y cálculo de latencias, diríjase al documento [Análisis Matemático del DAQ](docs/analisis_matematico.md).

## Estructura del Repositorio
*   `Core/`: Contiene los archivos fuente y de cabecera en lenguaje C, generados en cumplimiento con la capa de abstracción de hardware (HAL) de STMicroelectronics.
*   `docs/`: Documentación teórica, análisis de tiempos y especificaciones formales de la adquisición.
*   `scripts/`: Scripts en lenguaje Python para la decodificación del flujo binario a través de puertos asíncronos UART y reconstrucción de la señal en el dominio temporal para su posterior uso en algoritmos de identificación.

## Procedimiento de Uso
1.  Compilar el código fuente mediante STM32CubeIDE o un entorno con compilador GCC ARM y programar el microcontrolador.
2.  Verificar las conexiones físicas de los GPIOs al atenuador digital y a las salidas analógicas del LTC5596.
3.  Ejecutar el script `serial_plotter.py` en el ordenador receptor, asegurando la correspondencia del puerto COM y la tasa de baudios.
4.  El microcontrolador ejecutará automáticamente la perturbación transitoria y la transmisión del lote completo de muestras, revelando la curva de respuesta dinámica para el modelado del sistema físico.
