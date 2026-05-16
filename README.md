[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/rb0M7Pn8)
[![Open in Visual Studio Code](https://classroom.github.com/assets/open-in-vscode-2e0aaae1b6195c2367325f4f02e2d04e9abb55f0b24a779b69b11b9e10269abc.svg)](https://classroom.github.com/online_ide?assignment_repo_id=23907668&assignment_repo_type=AssignmentRepo)
# Lab07: Visualización en LCD 16x2 usando módulo I²C con microcontrolador PIC

## Integrantes

*[Steven Hinestroza Lopez]
(https://github.com/stevenh22)

*[Nicolas Andres Rodriguez Moreno]
(https://github.com/nicolasanrodriguezmo-dotcom)

## Documentación

En este laboratorio se implementó la comunicación entre el microcontrolador PIC18F45K22 y una pantalla LCD 16x2 utilizando el protocolo de comunicación serial I²C mediante el módulo expansor PCF8574. El objetivo principal fue reducir la cantidad de pines utilizados por la LCD, pasando de una conexión paralela tradicional de 6 u 8 líneas a únicamente dos líneas de comunicación: SDA y SCL. Para ello, se configuró el módulo MSSP del PIC en modo maestro I²C, permitiendo el envío de comandos y caracteres hacia la pantalla LCD a través del bus serial.

El funcionamiento del sistema se basa en el protocolo I²C, el cual utiliza una comunicación síncrona half-duplex entre el dispositivo maestro y los dispositivos esclavos conectados al mismo bus. En este caso, el PIC actúa como maestro mientras que el módulo PCF8574 funciona como esclavo, encargado de convertir los datos seriales recibidos en señales paralelas necesarias para el control de la LCD. La dirección utilizada para el dispositivo fue 0x4E, correspondiente a la dirección base 0x27 desplazada un bit hacia la izquierda para operaciones de escritura.


## Diagramas

## Evidencias de implementación

## Preguntas

1. ¿Por qué I²C se clasifica como half-duplex mientras que SPI es full-duplex? ¿Qué implicación práctica tiene esa diferencia para el control de una LCD?.

## 1.  I²C (half duplex) vs SPI (full duplex) y su implicacion en el control de LCD

Half-duplex significa que la línea SDA solo puede transportar datos en una dirección a la vez: o el maestro transmite al esclavo, o el esclavo responde al maestro, pero nunca ambos simultáneamente. En I²C esto es estructural, ya que solo existe una línea de datos compartida.Full-duplex en SPI es posible porque hay dos líneas de datos independientes: MOSI (Master Out, Slave In) y MISO (Master In, Slave Out), permitiendo enviar y recibir en el mismo ciclo de reloj.

2. En I2C_init() se asigna SSPCON1 = 0x28. Desglose ese valor bit a bit e identifique qué modo de operación del MSSP se está seleccionando y por qué se elige ese valor.

## 2. Desglose bit a bit de `SSPCON1 = 0x28`

`0x28` en binario es `0010 1000`.

| Bit | Nombre | Valor | Significado |
|:---:|:------:|:-----:|-------------|
| 7   | WCOL   | 0     | Sin colisión de escritura |
| 6   | SSPOV  | 0     | Sin desbordamiento del buffer |
| 5   | SSPEN  | 1     | **Habilita el módulo MSSP** (activa SDA y SCL) |
| 4   | CKP    | 0     | No aplica en modo I²C maestro |
| 3   | SSPM3  | 1     | Bit 3 de selección de modo |
| 2   | SSPM2  | 0     | Bit 2 de selección de modo |
| 1   | SSPM1  | 0     | Bit 1 de selección de modo |
| 0   | SSPM0  | 0     | Bit 0 de selección de modo → `SSPM[3:0] = 1000` |

Los bits `SSPM[3:0] = 1000` seleccionan el modo **I²C Maestro**, con velocidad
de reloj definida por `SSPADD` según la fórmula:

$$SSPADD = \frac{F_{osc}}{4 \times BaudRate} - 1$$

Se elige `0x28` porque es el valor que simultáneamente habilita el módulo
(`SSPEN = 1`) y lo configura como maestro I²C (`SSPM[3:0] = 1000`),
siendo los demás bits cero, correspondiente a su estado de reset limpio.

3. Las funciones I2C_start(), I2C_stop() e I2C_write() comparten el mismo patrón: activar un bit de control y luego esperar con while(!PIR1bits.SSPIF). ¿Qué representa la bandera SSPIF y por qué se limpia después de cada operación?.

## 3. La bandera `SSPIF` y por qué se limpia después de cada operación

`SSPIF` (*MSSP Interrupt Flag*, `PIR1bits.SSPIF`) es una bandera de hardware 
que el módulo MSSP pone en `1` automáticamente cada vez que completa una 
operación en el bus: un Start, un Stop, la transmisión de un byte, o la 
recepción de un ACK.

El patrón que comparten las tres funciones es el siguiente:

```c
// 1. Ordenar la operación
SSPCON2bits.SEN = 1;   // o PEN, RSEN, etc.

// 2. Esperar a que el hardware la complete
while(!PIR1bits.SSPIF);

// 3. Limpiar la bandera para la próxima operación
PIR1bits.SSPIF = 0;
```

**¿Por qué se limpia?** Porque `SSPIF` es una bandera *sticky*: el hardware 
la activa pero nunca la borra automáticamente. Si no se limpia manualmente 
antes de la siguiente operación, el `while(!PIR1bits.SSPIF)` de la función 
siguiente encontraría la bandera ya en `1` desde la operación anterior y no 
esperaría a que el hardware termine realmente, corrompiendo la secuencia de 
comunicación. Limpiarla garantiza que cada espera sea genuina y sincronizada 
con la operación actual.

4. El fuse PBADEN = OFF está presente en la configuración. ¿Qué efecto tendría dejarlo en ON sobre los pines del puerto B, y por qué podría causar problemas si se usan esos pines como salidas digitales?.

## 4. Efecto de fuse `PBADEN = ON` sobre el puerto B

El fuse PBADEN controla el comportamiento inicial de algunos pines del puerto B del PIC al momento del encendido o reinicio del microcontrolador.

Cuando PBADEN = ON, varios pines del puerto B se configuran automáticamente como entradas analógicas del módulo ADC. En este estado, los pines no funcionan correctamente como entradas o salidas digitales hasta que el software deshabilite manualmente la función analógica.

5. Compare el control de la LCD en modo paralelo (lab04) con el modo I²C de este laboratorio. Mencione ventajas y desventajas de cada enfoque en términos de: cantidad de pines usados, velocidad de actualización y complejidad del código.

## 5. LCD modo paralelo (Lab04) vs modo  I²C  

El modo paralelo utiliza varias líneas de datos y control para manejar directamente la pantalla LCD. Generalmente requiere entre 6 y 8 pines del microcontrolador, lo que incrementa el uso de recursos hardware. Sin embargo, este método ofrece una velocidad de actualización mayor, ya que los datos se transfieren directamente hacia la pantalla sin protocolos adicionales de comunicación.

Por otro lado, el modo I²C utiliza únicamente dos líneas (SDA y SCL) gracias al módulo expansor PCF8574. Esto representa una gran ventaja en proyectos donde se necesita ahorrar pines del microcontrolador o conectar múltiples dispositivos al mismo bus.

6. El bus I²C permite conectar múltiples esclavos con solo dos hilos. Si se quisiera agregar un segundo módulo PCF8574 al mismo bus (por ejemplo, para controlar un segundo LCD), ¿qué cambio mínimo sería necesario en el hardware y en el código?

## 6. Agregar un segundo PCF8574 al bus  I²C 

Cambio en hardware: El PCF8574 expone tres pines de configuración de dirección: A0, A1 y A2. La dirección base es 0x27 cuando los tres están en alto. Para el segundo módulo basta con cambiar al menos uno de esos pines a nivel bajo, por ejemplo, poner A0 a GND, lo que le asigna la dirección 0x26. Ambos módulos comparten físicamente las mismas líneas SDA y SCL; no se requiere ningún cable adicional más allá de conectar el segundo módulo al mismo bus.


## Conclusiones

1. El protocolo I²C reduce significativamente el uso de pines del microcontrolador. Mientras que el control paralelo de la LCD requiere entre 6 y 8 líneas de GPIO, la implementación con el módulo PCF8574 utiliza únicamente SCL y SDA, liberando recursos del PIC para otras tareas y simplificando el cableado del sistema.

2. El módulo MSSP del PIC18F45K22 ofrece un control robusto y flexible del bus I²C. La correcta configuración de los registros SSPCON1, SSPCON2 y SSPADD es fundamental para establecer la velocidad de comunicación y garantizar la correcta generación de las condiciones de Start, Stop y ACK, sin las cuales la transmisión falla silenciosamente.

3. La comprensión del direccionamiento I²C es esencial para evitar conflictos en el bus. El desplazamiento de la dirección base del PCF8574 (0x27 << 1 = 0x4E) refleja la convención del protocolo de transmitir 7 bits de dirección más el bit de lectura/escritura como un solo byte de 8 bits. Un error en este cálculo impide toda comunicación con el periférico.

4. La LCD en modo 4 bits vía I²C exige una secuencia de inicialización precisa. El envío correcto de los nibbles alto y bajo, junto con el control de la señal EN y los tiempos de retardo, determina el funcionamiento estable de la pantalla. Pequeñas variaciones en los tiempos o en el orden de las señales pueden provocar que la LCD quede en un estado indefinido.

5. La implementación de funciones modulares facilita la reutilización del código. Separar las capas de abstracción (I²C → PCF8574 → LCD) permite adaptar el código a distintas aplicaciones sin modificar la lógica de comunicación de bajo nivel, aplicando el principio de responsabilidad única en el diseño de software embebido.

## Referencias

[1] Microchip Technology Inc., PIC18(L)F2X/4XK22 Data Sheet, Microchip Technology, 2021.

[2] NXP Semiconductors, PCF8574 Remote 8-bit I/O Expander for I²C-Bus Data Sheet, NXP, 2013.

[3] M. Predko, Programming and Customizing PIC Microcontrollers, 3rd ed. New York, USA: McGraw-Hill, 2002.

[4] J. Iovine, PIC Microcontroller Project Book, New York, USA: McGraw-Hill Education, 2000.

[5] Microchip Technology Inc., MPLAB XC8 C Compiler User’s Guide, Microchip Technology, 2023.

[6] D. Ibrahim, Advanced PIC Microcontroller Projects in C, Oxford, U.K.: Newnes, 2008.

[7] P. Scherz and S. Monk, Practical Electronics for Inventors, 4th ed. New York, USA: McGraw-Hill Education, 2016.