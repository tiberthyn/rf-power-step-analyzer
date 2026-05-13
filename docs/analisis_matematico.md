# Análisis de Adquisición de Datos (TIM2 + ADC + DMA)

Para garantizar que la adquisición de señales refleje fielmente el comportamiento analógico (especialmente en transmisores de alta frecuencia como los de **5G mmWave de 25 GHz**) sin pérdida de muestras, el sistema entrelaza tres periféricos que operan de forma autónoma mediante hardware, liberando carga de procesamiento a la CPU.

---

## 1. Temporización: El Timer 2 como "Master Clock"
El **Timer 2 (TIM2)** actúa como el director de orquesta o marcapasos del sistema. En lugar de generar interrupciones que saturen la CPU, utiliza una señal de hardware denominada **TRGO (Trigger Output)**.

### Configuración del Reloj
* **Frecuencia del sistema ($f_{SYS}$):** $80 \text{ MHz}$ (Configurado vía PLL en `SystemClock_Config`).
* **Frecuencia de muestreo ($f_s$):** Determinada por el *Prescaler* y el *Periodo* (Auto-reload Register).

### Cálculo Matemático
La fórmula para la frecuencia de disparo del trigger es:
$$f_s = \frac{f_{SYS}}{(PSC + 1) \times (ARR + 1)}$$

Sustituyendo los valores del firmware:
* **Prescaler (PSC):** 79
* **Period (ARR):** 1

$$f_s = \frac{80,000,000}{(79 + 1) \times (1 + 1)} = \frac{80,000,000}{80 \times 2} = 500 \text{ kHz}$$

> [!NOTE]
> Esto resulta en un periodo de muestreo estricto de **$T_s = 2 \ \mu s$**, cumpliendo con la definición `#define FS_HZ 500000`.

---

## 2. Digitalización: ADC en Modo Escaneo (Scan Mode)
Cada vez que el `TIM2_TRGO` emite un pulso, el ADC se activa instantáneamente.

* **Configuración:** 2 canales, resolución de 12 bits.
* **Modo Escaneo:** El ADC lee secuencialmente el **Canal 8** (ej. entrada de potencia) y el **Canal 4** (ej. salida de potencia) en un solo ciclo de disparo.
* **Cuantificación:** La señal se mapea en un rango de $0$ a $4095$ ($2^{12}$ niveles discretos).

---

## 3. Transferencia: DMA y Gestión de Memoria
El **DMA (Direct Memory Access)** es el encargado de mover los datos del ADC a la RAM sin intervención del procesador.

* **Estructura del Buffer:** Los datos se almacenan de forma entrelazada en `adc_buffer`:
  `[CH8_0, CH4_0, CH8_1, CH4_1, ..., CH8_N, CH4_N]`
* **Estrategia de Captura:** La arquitectura utiliza un disparo único (**Single-Shot**) para capturar un bloque transitorio de **$N = 5000$** muestras.
* **Control de Flujo:** Al completarse la transferencia, la interrupción `HAL_ADC_ConvCpltCallback` detiene deliberadamente el Timer y el ADC para evitar que nuevos datos sobrescriban la captura transitoria.

---

## 4. Análisis de Holgura (Timing Slack)
Es crítico verificar si el ADC es capaz de procesar ambos canales antes de que llegue el siguiente pulso del Timer.

### Tiempo Disponible ($T_{available}$)
Como $f_s = 500 \text{ kHz}$, el tiempo entre disparos es:
$$T_{trigger} = \frac{1}{500,000 \text{ Hz}} = 2.0 \ \mu s$$

### Tiempo de Conversión del ADC ($T_{conv\_total}$)
El tiempo de conversión para un canal es la suma del tiempo de muestreo y el tiempo de resolución (12.5 ciclos para 12 bits):
$$T_{ch} = \text{Sample Time} + 12.5 \text{ ciclos}$$

1. **Canal 8:** Configurado a `ADC_SAMPLETIME_47CYCLES_5`. Total = $47.5 + 12.5 = 60 \text{ ciclos}$.
2. **Canal 4:** Configuración idéntica. Total = $60 \text{ ciclos}$.
3. **Total Scan (2 canales):** $120 \text{ ciclos de reloj del ADC}$.

Si el reloj del ADC ($f_{ADC}$) corre a $80 \text{ MHz}$:
$$T_{conv\_total} = \frac{120 \text{ ciclos}}{80,000,000 \text{ Hz}} = 1.5 \ \mu s$$

### Cálculo de la Holgura (Slack)
$$\text{Holgura} = T_{trigger} - T_{conv\_total} = 2.0 \ \mu s - 1.5 \ \mu s = \mathbf{0.5 \ \mu s}$$

> [!IMPORTANT]
> Existe un **25% de margen de seguridad**. El sistema es estable y no hay riesgo de colisión de muestras (*overrun*).

---

## 5. Optimización y Límites

1. **Reducción del Tiempo de Muestreo:** Podrías bajar a `ADC_SAMPLETIME_12CYCLES_5`.
   * *Riesgo:* Si los detectores **LTC5596** tienen alta impedancia de salida, bajar los ciclos distorsionará la lectura debido a una carga incompleta del capacitor de muestreo.
2. **Límite Físico:** El límite teórico de frecuencia para esta configuración de 120 ciclos es:
   $$f_{s\_max} = \frac{1}{1.5 \ \mu s} \approx 666 \text{ kHz}$$

### Recomendación Técnica
Dada la aplicación en **control de potencia 5G**, la holgura de $0.5 \ \mu s$ es valiosa para absorber pequeñas latencias en el bus AHB. Se recomienda mantener los ciclos de muestreo actuales para asegurar la integridad de la señal.
