# Análisis Matemático de la Cadena de Adquisición (DAQ)

El sistema de adquisición de datos en el transmisor 5G mmWave de 25 GHz está fundamentado en una arquitectura de hardware autónoma. Esta topología entrelaza tres periféricos críticos (TIM2, ADC1, DMA1) para garantizar un **determinismo temporal estricto** sin la intervención de la Unidad Central de Procesamiento (CPU), con el objetivo fundamental de capturar fielmente la **respuesta transitoria** para la **identificación del sistema**.

---

## 1. Arquitectura de Sincronización y Temporización

El temporizador **TIM2** opera como el generador de la señal de disparo maestro (TRGO). La frecuencia de actualización, que dicta el periodo de muestreo, se determina mediante la frecuencia del bus del sistema ($f_{SYS}$), el pre-escalador ($PSC$) y el periodo de auto-recarga ($ARR$).

Dada una configuración del PLL que establece la frecuencia del sistema en **80 MHz**, los parámetros definidos en el firmware ($PSC = 79$, $ARR = 1$) producen la siguiente frecuencia de interrupción ($f_s$):

$$f_s = \frac{f_{SYS}}{(PSC + 1) \times (ARR + 1)}$$

$$f_s = \frac{80,000,000}{(79 + 1) \times (1 + 1)} = 500,000 \text{ Hz}$$

> [!NOTE]
> El periodo de muestreo estricto resultante es $T_s = \mathbf{2.0 \ \mu s}$, asegurando una base de tiempo constante y confiable para la correcta identificación paramétrica del sistema.

---

## 2. Tiempos de Conversión Analógico-Digital

El conversor **ADC1** opera en *Scan Mode* secuencial para dos canales (Canal 8 para entrada de potencia y Canal 4 para salida) con una resolución de 12 bits. El tiempo total de conversión ($T_{conv\_total}$) es la suma de los ciclos requeridos para el muestreo de la capacitancia interna y los ciclos de aproximación sucesiva.

Para el Canal 8, configurado con `ADC_SAMPLETIME_47CYCLES_5`, el tiempo en ciclos de reloj de ADC ($C_{ch8}$) es:

$$C_{ch8} = 47.5 \text{ (muestreo)} + 12.5 \text{ (resolución)} = 60 \text{ ciclos}$$

Asumiendo idéntica parametrización para el Canal 4, el requisito total para un barrido completo es de **120 ciclos**. Con un reloj de ADC derivado sincrónicamente a **80 MHz**, el tiempo físico de conversión es:

$$T_{conv\_total} = \frac{120}{80,000,000} = 1.5 \ \mu\text{s}$$

---

## 3. Transferencia de Datos (DMA) y Captura del Transitorio

Para evitar que la CPU invierta ciclos de reloj leyendo los registros del ADC, el **DMA1** (Acceso Directo a Memoria) traslada automáticamente cada muestra al `adc_buffer` en la memoria RAM. 

Se utiliza una arquitectura de disparo único (*Single-Shot*) para capturar un bloque definido de datos transitorios. Al completarse la transferencia de las muestras, la interrupción del DMA desactiva el TIM2 y el ADC. Esto congela la captura y protege la integridad del búfer contra sobreescrituras, permitiendo extraer la curva de respuesta al escalón intacta para aplicarle algoritmos de modelado e identificación de sistemas.

---

## 4. Análisis de Holgura Temporal (Slack Time)

Para asegurar que no exista solapamiento de datos ni condiciones de carrera en el bus AHB durante la transferencia del DMA, es imperativo que el tiempo de conversión sea estrictamente menor al periodo de muestreo. 

La holgura temporal ($\Delta t_{slack}$) del sistema se define como:

$$\Delta t_{slack} = T_s - T_{conv\_total}$$

$$\Delta t_{slack} = 2.0 \ \mu\text{s} - 1.5 \ \mu\text{s} = 0.5 \ \mu\text{s}$$

> [!IMPORTANT]
> **Estabilidad del Sistema:** El DAQ presenta una holgura del **25%** respecto al ciclo de trabajo de la señal de disparo. Esto demuestra estabilidad matemática para operar a **500 kHz** sin pérdida de tramas (*frame drops*) ni colisiones en memoria durante el transitorio.

---

## 5. Integridad de Señal e Impedancia

Aunque matemáticamente es posible reducir $T_{conv\_total}$ disminuyendo los ciclos de muestreo del ADC (por ejemplo, a **12.5** ciclos) para aumentar la frecuencia de adquisición, esta práctica está contraindicada.

> [!WARNING]
> **Riesgo de Distorsión en el Modelado:** Una reducción adicional del tiempo de muestreo podría introducir distorsión armónica y lecturas erróneas. Si la impedancia de salida de los detectores **LTC5596** interactúa negativamente con tiempos de carga de la red capacitiva interna del ADC inferiores a **47.5 ciclos**, el capacitor no alcanzará el voltaje real de la envolvente de RF, lo que corrompería los datos capturados y resultaría en un modelo matemático erróneo del sistema.
