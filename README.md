# mqtt-esp32-security

Vamos a analizar la seguridad del protocolo MQTT.

Para esta prueba de concepto, se ha creado un topic denominado `auditoria/esp32/datos`, el cual opera sin protección TLS (puerto 1883) y sin autenticación (sin usuario ni contraseña).

## Código del ESP32

```cpp
#include <WiFi.h>
#include <PubSubClient.h>

// 1. Configuración de Red
const char* ssid = "ssid";
const char* password = "passwd";

// 2. Configuración MQTT (Sin usuario/contraseña)
const char* mqtt_server = "broker.emqx.io";
const int mqtt_port = 1883;
const char* mqtt_topic = "auditoria/esp32/datos"; // Usa un topic único

WiFiClient espClient;
PubSubClient client(espClient);

unsigned long lastMsg = 0;
int valor = 10;

void setup_wifi() {
  delay(10);
  Serial.println();
  Serial.print("Conectando a ");
  Serial.println(ssid);
  
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  
  Serial.println("\nWiFi conectado.");
  Serial.print("Dirección IP: ");
  Serial.println(WiFi.localIP());
}

void reconnect() {
  // Bucle hasta que estemos conectados
  while (!client.connected()) {
    Serial.print("Intentando conexión MQTT...");
    
    // Generar un ID de cliente aleatorio
    String clientId = "ESP32Auditor-" + String(random(0xffff), HEX);
    
    // Intentar conectar (solo con el ID, sin credenciales)
    if (client.connect(clientId.c_str())) {
      Serial.println("Conectado al broker!");
    } else {
      Serial.print("Fallo, rc=");
      Serial.print(client.state());
      Serial.println(" reintento en 5 segundos");
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  randomSeed(micros());
  
  setup_wifi();
  client.setServer(mqtt_server, mqtt_port);
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  unsigned long now = millis();
  // Publicar cada 3000 ms (3 segundos)
  if (now - lastMsg > 3000) {
    lastMsg = now;
    
    // Convertir el número a texto para enviarlo
    String payload = String(valor);
    
    Serial.print("Publicando en ");
    Serial.print(mqtt_topic);
    Serial.print(": ");
    Serial.println(payload);
    
    client.publish(mqtt_topic, payload.c_str());
    
    // Incrementar en 10 para el siguiente ciclo
    valor += 10; 
  }
}
```

## Auditoría desde terminal en Kali Linux

En primer lugar, actualizamos los repositorios e instalamos los clientes de Mosquitto y la herramienta de captura de red.

```bash
sudo apt update && sudo apt install mosquitto-clients tcpdump
```

<img width="872" height="530" alt="image" src="https://github.com/user-attachments/assets/eccd17c2-1016-493d-9f29-5bc506acb5b9" />

### Suscripción al topic 

Nos suscribimos al topic `"auditoria/esp32/datos"` para interceptar la telemetría en texto plano:

```bash
mosquitto_sub -h broker.emqx.io -p 1883 -t "auditoria/esp32/datos" -d
```

<img width="773" height="342" alt="image" src="https://github.com/user-attachments/assets/41740bba-444f-4447-93f2-92f08b0f0a49" />

Como se aprecia arriba, nos hemos suscrito al topic y somos capaces de ver lo que el ESP32 está publicando.

<img width="1919" height="489" alt="image" src="https://github.com/user-attachments/assets/63401d2f-eb1b-46de-9f73-1fc352cdfe3d" />

### Inyección de datos

Como hemos comentado antes, el topic no tiene usuario ni contraseña. ¿Qué pasaría si alguien externo quisiera introducir información errónea? ¿Sería posible? Lo vemos a continuación.

Con el siguiente comando inyectaremos el valor `"666"` en el flujo de datos:

```bash
mosquitto_pub -h broker.emqx.io -p 1883 -t "auditoria/esp32/datos" -m "666" -d
```

<img width="1917" height="731" alt="image" src="https://github.com/user-attachments/assets/369a883b-eae3-471d-8308-a0f93ec813e9" />

**Resultado:** Hemos podido mandar información errónea al topic con éxito, demostrando que el canal es vulnerable a ataques de inyección si no se implementa autenticación.
