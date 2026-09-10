# 📘 MANUAL DE PRÁCTICAS DE LABORATORIO
## Redes de Telecomunicaciones e IoT: Implementación de Zigbee en Entorno Arduino
---
**Institución:** Instituto Tecnológico Superior del Occidente del Estado de Hidalgo  
**División:** Ingeniería en Tecnologías de la Información y Comunicaciones  
**Elaborado por:** Prof. Saúl Isaí Soto Ortiz  

---

## 📖 Introducción General
El protocolo Zigbee (basado en el estándar IEEE 802.15.4) es un pilar en la automatización industrial gracias a su topología de malla (Mesh) y bajo consumo energético. Esta serie de prácticas guiará al alumno en la configuración de una red Zigbee utilizando el microcontrolador **ESP32-C6**, programado mediante el **Arduino Core**.

Todas las prácticas están diseñadas para simularse en el entorno **Wokwi**, superando las limitaciones físicas al abstraer la comunicación de radiofrecuencia a un nivel lógico dentro del simulador.

---

## 🛠 Consideraciones para el Simulador (Wokwi y Arduino)
El simulador Wokwi compila un solo archivo `sketch.ino` para todas las placas del proyecto. Para lograr que dos o más ESP32-C6 asuman roles distintos (ej. uno como Coordinador y otro como End Device) ejecutando el mismo código, utilizaremos una técnica de **Selección de Rol por Hardware (Hardware Switch)**.

1. **Un solo proyecto:** Todos los nodos deben existir en el mismo archivo `diagram.json`.
2. **Éter Virtual:** No se requieren conexiones físicas entre las antenas.
3. **El Hardware Switch:** Utilizaremos el pin `GPIO4`. Si en el esquema lógico conectamos el pin `GPIO4` a `GND`, la placa asumirá el rol de Coordinador. Si lo dejamos desconectado, asumirá el rol de End Device.

---
---

# 🔬 PRÁCTICA 1: Formación de Red y Selección de Rol (Commissioning)

### 🎯 Objetivo de la Práctica
Comprender la estructura base de una red Zigbee configurando un dispositivo central (Coordinador) y un periférico (End Device) utilizando lectura de pines digitales para definir la jerarquía al arranque.

### 🔌 Esquema de Conexión Lógica (diagram.json)
| Dispositivo Wokwi | Rol en la Red | Configuración Física del Hardware Switch |
| :--- | :--- | :--- |
| `ESP32-C6 (Nodo A)` | Coordinador (ZC) | Pin `GPIO4` conectado a `GND`. |
| `ESP32-C6 (Nodo B)` | End Device (ZED) | Pin `GPIO4` desconectado (flotante). |

### 💻 Implementación en Código (`sketch.ino`)

Código base para la gestión de roles en el `setup()` y procesamiento en el `loop()`:

    #include <Arduino.h>
    #include "esp_zigbee_core.h"
    
    #define ROLE_PIN 4
    
    void setup() {
        Serial.begin(115200);
        pinMode(ROLE_PIN, INPUT_PULLUP);
        delay(100); // Estabilizar voltaje
        
        if (digitalRead(ROLE_PIN) == LOW) {
            Serial.println("Iniciando como COORDINADOR (ZC)...");
            esp_zb_cfg_t zb_nwk_cfg = ESP_ZB_ZC_CONFIG();
            esp_zb_init(&zb_nwk_cfg);
            esp_zb_set_primary_network_channel_set(ESP_ZB_TRANSCEIVER_MAC_CHANNEL_15);
            esp_zb_start(false);
        } else {
            Serial.println("Iniciando como END DEVICE (ZED)...");
            esp_zb_cfg_t zb_nwk_cfg = ESP_ZB_ZED_CONFIG();
            esp_zb_init(&zb_nwk_cfg);
            esp_zb_set_primary_network_channel_set(ESP_ZB_TRANSCEIVER_MAC_CHANNEL_15);
            esp_zb_start(false);
        }
    }
    
    void loop() {
        // Procesar eventos de red continuamente
        esp_zb_main_loop_iteration();
        delay(10); // Ceder tiempo al RTOS
    }

### ✅ Resultados a Evaluar
Al iniciar, la consola del Nodo A imprimirá su inicialización como ZC y formará la red. El Nodo B imprimirá su arranque como ZED y reportará `Joined network successfully`.

### 🧠 Análisis Crítico y Evidencia Práctica (Obligatorio)
1. **Prueba de aislamiento de red:** Modifica el código del ZED (dentro del bloque `else`) para que escuche exclusivamente en el canal 20, mientras el ZC permanece en el 15. Ejecuta la simulación. 
   * Pega aquí el mensaje de error o *warning* que arroja la terminal del ZED.
   * Explica con tus propias palabras por qué el stack MAC abortó la conexión.
2. **Reinicio abrupto:** Con la red funcionando en el canal 15, presiona el botón de reinicio (Reset) ÚNICAMENTE en el Nodo A (Coordinador). 
   * ¿Qué mensajes imprime el Nodo B al detectar la ausencia de su padre lógico? Documenta tu observación.

---
---

# 🔬 PRÁCTICA 2: Endpoints y Direccionamiento Unicast

### 🎯 Objetivo de la Práctica
Estructurar la capa de aplicación transmitiendo lecturas de telemetría punto a punto (Unicast) integrando el flujo de Arduino.

### 🧠 Marco Conceptual Ampliado
*   **Direccionamiento Corto:** Cada nodo recibe una dirección de 16 bits. El Coordinador siempre es `0x0000`.
*   **Endpoints:** Puertos virtuales (1-254) que definen una aplicación específica (ej. Temperatura).

### 🔌 Esquema de Conexión Física/Lógica
| Dispositivo Wokwi | Rol | Conexiones Físicas Adicionales |
| :--- | :--- | :--- |
| `ESP32-C6 (Nodo A)` | ZC | `GPIO4` a `GND` |
| `ESP32-C6 (Nodo B)` | ZED | `GPIO4` desconectado + **Sensor DHT22 en GPIO5** |

### 💻 Implementación en Código

Creación de un Endpoint en el bloque del End Device (`else` en el `setup()`):

    // Crear la lista de Endpoints del dispositivo
    esp_zb_ep_list_t *ep_list = esp_zb_ep_list_create();
    
    // Configurar el Endpoint número 10 para domótica
    esp_zb_endpoint_config_t ep_config = {
        .endpoint = 10, 
        .app_profile_id = ESP_ZB_AF_HA_PROFILE_ID, 
    };
    
    // Registrar el clúster de temperatura
    esp_zb_ep_list_add_ep(ep_list, esp_zb_create_custom_ep(&ep_config));

### ✅ Resultados a Evaluar
El ZED leerá el DHT22 usando librerías de Arduino y enviará un paquete Unicast a la dirección `0x0000`. La consola del ZC imprimirá la recepción.

### 🧠 Análisis Crítico y Evidencia Práctica (Obligatorio)
1. **Fallo de Enrutamiento Inducido:** Modifica la función de envío en el ZED para que apunte a la dirección `0x1234` en lugar de `0x0000`. 
   * Captura la salida de la consola del ZED. ¿Qué protocolo interno intentó usar Zigbee para encontrar esa dirección (Route Request)?
2. **Uso de delays en Arduino:** Si colocamos un `delay(5000);` dentro de la función `loop()`, ¿qué impacto tiene sobre la función `esp_zb_main_loop_iteration()` y la estabilidad de la red?

---
---

# 🔬 PRÁCTICA 3: Topología Mesh y Enrutamiento

### 🎯 Objetivo de la Práctica
Comprender el concepto de "saltos" (hops) añadiendo un tercer estado a nuestro Hardware Switch para instanciar un Router Zigbee (ZR).

### 🔌 Esquema de Conexión Lógica
Ampliaremos nuestra lógica de `sketch.ino` leyendo dos pines (`GPIO4` y `GPIO5`) para permitir 3 combinaciones.

| Dispositivo Wokwi | Rol en la Red | Configuración Pines (GPIO4 / GPIO5) |
| :--- | :--- | :--- |
| `ESP32-C6 (Nodo A)` | Coordinador (ZC) | `GND` / `GND` |
| `ESP32-C6 (Nodo B)` | Router (ZR) | `GND` / `Desconectado` |
| `ESP32-C6 (Nodo C)` | End Device (ZED) | `Desconectado` / `Desconectado` |

### 💻 Implementación en Código

Lógica de selección de 3 roles en el `setup()`:

    int pin4 = digitalRead(4);
    int pin5 = digitalRead(5);
    
    if (pin4 == LOW && pin5 == LOW) {
        // Inicializar como ZC
    } else if (pin4 == LOW && pin5 == HIGH) {
        // Inicializar como ZR
        esp_zb_cfg_t zb_nwk_cfg = ESP_ZB_ZR_CONFIG();
        esp_zb_init(&zb_nwk_cfg);
        esp_zb_set_primary_network_channel_set(ESP_ZB_TRANSCEIVER_MAC_CHANNEL_15);
        esp_zb_start(false);
    } else {
        // Inicializar como ZED
    }

### ✅ Resultados a Evaluar
El alumno utilizará comandos de consola para imprimir la tabla de enrutamiento y visualizará la dependencia jerárquica.

### 🧠 Análisis Crítico y Evidencia Práctica (Obligatorio)
1. **Análisis de Tablas en Tiempo Real:** Configura los tres nodos. Imprime la tabla de enrutamiento del Coordinador. Detén la simulación, borra el Nodo B (Router) del `diagram.json` y vuelve a correr.
   * Pega la nueva tabla. Explica matemáticamente cómo se reasignaron las direcciones lógicas.
2. **Consumo y Roles:** ¿Por qué sería un error de ingeniería usar baterías para alimentar el Nodo B (Router) en una implementación real?

---
---

# 🔬 PRÁCTICA 4: Control Grupal Eficiente (Broadcast)

### 🎯 Objetivo de la Práctica
Evaluar el impacto en el ancho de banda controlando actuadores múltiples (LEDs) con un solo mensaje al aire.

### 🔌 Esquema de Conexión Física
| Dispositivo Wokwi | Rol y Periférico | Descripción |
| :--- | :--- | :--- |
| `Nodo A` | ZC + Push Button | Botón en `GPIO9` (resistencia pull-up) |
| `Nodo B` | ZED 1 + LED | LED Rojo en `GPIO2` |
| `Nodo C` | ZED 2 + LED | LED Azul en `GPIO2` |

### 💻 Implementación en Código

Envío de Broadcast desde el Coordinador al detectar el botón en el `loop()`:

    if (digitalRead(9) == LOW) {
        esp_zb_zcl_on_off_cmd_t cmd_req;
        cmd_req.zcl_basic_cmd.src_endpoint = 1;
        cmd_req.address_mode = ESP_ZB_APS_ADDR_MODE_16_GROUP_ENDP_NOT_PRESENT;
        
        // 0xFFFF indica a la red que es un Broadcast
        cmd_req.zcl_basic_cmd.dst_addr_u.addr_short = 0xFFFF; 
        cmd_req.on_off_cmd_id = ESP_ZB_ZCL_CMD_ON_OFF_TOGGLE_ID;
        
        esp_zb_zcl_on_off_cmd_req(&cmd_req);
        delay(300); // Antirrebote básico
    }

### ✅ Resultados a Evaluar
Al presionar el botón en el ZC, ambos LEDs (Nodos B y C) deberán cambiar su estado simultáneamente con un solo paquete TX.

### 🧠 Análisis Crítico y Evidencia Práctica (Obligatorio)
1. **Saturación del Canal (Broadcast Storm):** Elimina la condicional del botón y el `delay(300)`, forzando al Coordinador a enviar el Broadcast en cada ciclo del `loop()`.
   * Pega el error de desbordamiento (Out of Memory) que lanza la consola al saturar el buffer de radio.
2. **Eficiencia:** Si cambiamos el Broadcast por 2 comandos Unicast consecutivos, describe la diferencia en eficiencia espectral.

---
---

# 🔬 PRÁCTICA 5: Concurrencia Segura con FreeRTOS en Arduino

### 🎯 Objetivo de la Práctica
Garantizar la estabilidad operativa integrando lecturas lentas y la radio Zigbee de forma asíncrona mediante el uso de tareas de FreeRTOS directamente en Arduino.

### 🧠 Marco Conceptual Ampliado
El SDK de Zigbee no es "Thread-Safe". Si el `loop()` principal de Arduino se bloquea calculando matemáticas de un sensor, Zigbee pierde la conexión. Usaremos `xTaskCreate` para mover el sensor a un núcleo/hilo secundario, y `xQueue` para pasar los datos al `loop()`.

### 💻 Implementación en Código

    #include <Arduino.h>
    #include "freertos/queue.h"
    
    QueueHandle_t sensor_queue;
    
    // Tarea secundaria para no bloquear Zigbee
    void sensor_task(void *pvParameters) {
        while(1) {
            float data = analogRead(A0); // Simulación de carga
            xQueueSend(sensor_queue, &data, portMAX_DELAY);
            vTaskDelay(pdMS_TO_TICKS(1000));
        }
    }
    
    void setup() {
        Serial.begin(115200);
        sensor_queue = xQueueCreate(5, sizeof(float));
        
        // Iniciar la tarea del sensor
        xTaskCreate(sensor_task, "Sensor", 2048, NULL, 1, NULL);
        
        // ... (Inicialización Zigbee según Hardware Switch) ...
    }
    
    void loop() {
        esp_zb_main_loop_iteration();
        
        float rx_data;
        // Leer la cola. Timeout 0 es CRÍTICO.
        if(xQueueReceive(sensor_queue, &rx_data, 0) == pdTRUE) {
            // Actualizar cluster Zigbee de forma segura
        }
        delay(10);
    }

### ✅ Resultados a Evaluar
El sistema se mantiene en línea estable. Los retardos intencionales agregados a la `sensor_task` no desconectan al dispositivo de la red Zigbee.

### 🧠 Análisis Crítico y Evidencia Práctica (Obligatorio)
1. **Provocando un Crash intencional:** Modifica `xQueueReceive(..., 0)` cambiando el parámetro a `portMAX_DELAY`. Haz que la `sensor_task` tarde 10 segundos en enviar un dato.
   * Pega el texto rojo de error (Watchdog) que lanza la consola de Wokwi.
   * Explica por qué FreeRTOS reinició el microcontrolador al aplicar esta espera bloqueante dentro del `loop()` principal.
