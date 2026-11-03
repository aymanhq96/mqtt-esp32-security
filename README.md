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
