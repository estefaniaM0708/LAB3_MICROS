# Lectura del sensor MPU6050 mediante ESP32

## Descripción del proyecto

En este proyecto se implementó un sistema de adquisición de datos utilizando el sensor inercial MPU6050 conectado a un microcontrolador ESP32 mediante comunicación I2C.

El MPU6050 integra un acelerómetro de tres ejes, un giroscopio de tres ejes y un sensor interno de temperatura, permitiendo obtener información relacionada con movimiento, orientación y condiciones térmicas del dispositivo.

El sistema fue desarrollado mediante dos metodologías de programación:

- Implementación utilizando Arduino IDE con lenguaje C/C++.
- Implementación utilizando MicroPython.

Ambas versiones realizan la misma función: adquirir los datos del sensor a una frecuencia de 20 Hz, procesarlos y enviarlos mediante comunicación serial para su posterior análisis.

---

# Configuración del hardware

## Conexión ESP32 - MPU6050

| MPU6050 | ESP32 |
|--------|-------|
| VCC | 3.3 V |
| GND | GND |
| SDA | GPIO 21 |
| SCL | GPIO 22 |

La comunicación entre ambos dispositivos se realiza mediante el protocolo I2C utilizando la dirección:

```
0x68
```

correspondiente al MPU6050.

---

# Funcionamiento general del sistema

El proceso de adquisición desarrollado en ambas implementaciones sigue la misma secuencia:

1. Inicialización del puerto I2C.
2. Configuración del MPU6050 para salir del modo de suspensión.
3. Lectura de los registros internos del sensor.
4. Adquisición de los 14 bytes correspondientes a:

   - Aceleración en X.
   - Aceleración en Y.
   - Aceleración en Z.
   - Temperatura.
   - Velocidad angular en X.
   - Velocidad angular en Y.
   - Velocidad angular en Z.

5. Conversión de datos digitales a unidades físicas.
6. Envío de información mediante puerto serial.

La frecuencia de muestreo utilizada es:

```
20 Hz
```

equivalente a una lectura cada:

```
50 ms
```

---

# Implementación Arduino IDE

## Descripción

La primera implementación fue desarrollada utilizando Arduino IDE mediante lenguaje C/C++.

Para la comunicación con el sensor se utiliza la librería:

```cpp
#include <Wire.h>
```

Esta librería permite realizar la comunicación I2C entre el ESP32 y el MPU6050 mediante escritura y lectura directa de registros.

---

## Configuración del MPU6050

Antes de iniciar la adquisición, el sensor debe salir del modo de suspensión.

Esto se realiza escribiendo un valor cero en el registro:

```
PWR_MGMT_1 (0x6B)
```

Código utilizado:

```cpp
Wire.beginTransmission(MPU);
Wire.write(0x6B);
Wire.write(0);
Wire.endTransmission();
```

---

## Temporización mediante interrupción

Para garantizar una frecuencia constante de adquisición se utiliza un temporizador hardware del ESP32:

```cpp
hw_timer_t *miTimer = NULL;
```

El temporizador se configura para generar una interrupción cada:

```
50000 microsegundos = 50 ms
```

Dentro de la interrupción solamente se activa una bandera:

```cpp
void IRAM_ATTR cadaCiclo() {
    listo = true;
}
```

La lectura del sensor no se realiza dentro de la interrupción, evitando retrasos por comunicación I2C.

---

## Lectura del sensor

El MPU6050 entrega los datos almacenados en registros consecutivos desde:

```
ACCEL_XOUT_H (0x3B)
```

Se solicitan 14 bytes:

```cpp
Wire.requestFrom(MPU,14,true);
```

Estos datos contienen:

| Registro | Variable |
|-|-|
| ACCEL_XOUT | Ax |
| ACCEL_YOUT | Ay |
| ACCEL_ZOUT | Az |
| TEMP_OUT | Temperatura |
| GYRO_XOUT | Gx |
| GYRO_YOUT | Gy |
| GYRO_ZOUT | Gz |

Los valores son almacenados como variables enteras de 16 bits:

```cpp
int16_t axRaw;
int16_t ayRaw;
int16_t azRaw;
```

---

## Conversión de datos

Los factores utilizados corresponden a la configuración por defecto del MPU6050:

### Acelerómetro

Sensibilidad:

```
16384 LSB/g
```

Conversión:

```cpp
ax = axRaw / 16384.0;
```

---

### Giroscopio

Sensibilidad:

```
131 LSB/(°/s)
```

Conversión:

```cpp
gx = gxRaw / 131.0;
```

---

### Temperatura

Ecuación:

```cpp
temp = tempRaw / 340.0 + 36.53;
```

---

# Implementación MicroPython

## Descripción

La segunda implementación fue desarrollada utilizando MicroPython sobre el ESP32.

A diferencia de Arduino IDE, MicroPython permite realizar la configuración del sensor utilizando funciones de más alto nivel, reduciendo la cantidad de código necesario.

Las librerías utilizadas son:

```python
from machine import Pin, I2C, Timer
import struct
import time
```

---

# Configuración I2C en MicroPython

El bus I2C se configura mediante:

```python
i2c = I2C(
    0,
    sda=Pin(21),
    scl=Pin(22),
    freq=400000
)
```

Manteniendo la misma conexión utilizada en Arduino IDE.

---

# Activación del MPU6050

El sensor se activa escribiendo en el registro:

```
0x6B
```

mediante:

```python
i2c.writeto_mem(
    MPU,
    0x6B,
    b'\x00'
)
```

---

# Control temporal

La adquisición se realiza mediante el módulo:

```python
Timer()
```

Configurado con un periodo de:

```
50 ms
```

Ejemplo:

```python
miTimer.init(
    period=50,
    mode=Timer.PERIODIC,
    callback=cadaCiclo
)
```

Al igual que en Arduino, la interrupción únicamente activa una bandera y la lectura se realiza posteriormente en el programa principal.

---

# Lectura del MPU6050

En MicroPython se utiliza:

```python
i2c.readfrom_mem()
```

para obtener directamente los 14 bytes del sensor:

```python
datos = i2c.readfrom_mem(
    MPU,
    0x3B,
    14
)
```

Posteriormente los datos se organizan mediante:

```python
struct.unpack(">hhhhhhh",datos)
```

obteniendo:

```python
axRaw
ayRaw
azRaw
tempRaw
gxRaw
gyRaw
gzRaw
```

---

# Comparación entre Arduino IDE y MicroPython

| Característica | Arduino IDE | MicroPython |
|-|-|-|
| Lenguaje | C/C++ | Python |
| Nivel de programación | Bajo nivel | Alto nivel |
| Comunicación I2C | Wire.h | Clase I2C |
| Acceso al sensor | Manipulación directa de registros | Funciones de memoria |
| Conversión de bytes | Manual | Uso de struct |
| Temporizador | hw_timer_t | Timer |
| Frecuencia de adquisición | 20 Hz | 20 Hz |
| Lectura realizada | 14 bytes | 14 bytes |
| Control del hardware | Mayor | Menor |
| Facilidad de desarrollo | Media | Alta |
| Optimización | Mayor | Menor |

---

# Diferencias principales

La implementación en Arduino IDE proporciona un control más detallado sobre los recursos internos del ESP32. Al trabajar directamente con C/C++, es posible realizar una administración más eficiente de memoria, interrupciones y periféricos.

Esta característica hace que Arduino IDE sea adecuado para aplicaciones donde se requiere alta precisión temporal y optimización del hardware.

Por otro lado, MicroPython utiliza una capa de abstracción superior que simplifica la programación del sistema. La comunicación I2C y el manejo de registros requieren menos líneas de código, facilitando el desarrollo y modificación del programa.

Aunque MicroPython sacrifica parte del rendimiento debido a que es un lenguaje interpretado, resulta una alternativa adecuada para prototipos, pruebas experimentales y sistemas donde la rapidez de desarrollo es importante.

---

# Resultados obtenidos

Ambas implementaciones permiten obtener correctamente:

- Aceleración en los tres ejes:

```
g
```

- Velocidad angular:

```
°/s
```

- Temperatura interna:

```
°C
```

Los datos enviados por puerto serial tienen el siguiente formato:

```
ax,ay,az,gx,gy,gz,temp
```

Ejemplo:

```
0.0123,-0.0251,0.9984,0.1527,-0.0916,0.0305,27.64
```

---

# Conclusión

Las dos implementaciones desarrolladas permiten utilizar correctamente el sensor MPU6050 con el ESP32 mediante comunicación I2C.

La versión realizada en Arduino IDE ofrece mayor control sobre el hardware y mejor capacidad de optimización, mientras que la versión en MicroPython facilita la programación y permite desarrollar prototipos funcionales con una estructura más sencilla.

A nivel funcional, ambas soluciones presentan el mismo comportamiento, realizando la adquisición de datos a 20 Hz y entregando las variables físicas necesarias para el análisis del movimiento del dispositivo.
