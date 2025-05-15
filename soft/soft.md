#include <ModbusMaster.h>
#include <WiFi.h>
#include <PubSubClient.h>

// --- Config WiFi ---
const char* ssid = "EB110 2.4Ghz";
const char* password = "EB110-direct";

// --- Config MQTT ---
const char* mqtt_server = "192.168.50.139";
const int mqtt_port = 1883;
const char* mqtt_user = "hsdell104";
const char* mqtt_pass = "104@eb104";

// --- Obiecte ---
WiFiClient espClient;
PubSubClient client(espClient);
ModbusMaster node;

#define MAX485_RE_DE 4

// Reîncercare la 10 secunde
unsigned long lastReconnectAttempt = 0;
const unsigned long reconnectInterval = 10000;

void preTransmission() {
  digitalWrite(MAX485_RE_DE, HIGH);
}

void postTransmission() {
  digitalWrite(MAX485_RE_DE, LOW);
}

void setup_wifi() {
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  Serial.print("Conectare WiFi...");
  int retries = 0;
  while (WiFi.status() != WL_CONNECTED && retries < 30) {
    delay(500);
    Serial.print(".");
    retries++;
  }
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println(" Conectat: " + WiFi.localIP().toString());
  } else {
    Serial.println("\nEșec WiFi. Continuare în mod reconectare automată.");
  }
}

bool reconnect_mqtt() {
  Serial.print("Conectare MQTT...");
  if (client.connect("ESP32_Janitza", mqtt_user, mqtt_pass)) {
    Serial.println(" Conectat la MQTT");
    return true;
  } else {
    Serial.print(" Eroare MQTT. Cod: ");
    Serial.println(client.state());
    return false;
  }
}

float readFloat(uint16_t address) {
  uint8_t result = node.readHoldingRegisters(address, 2);
  if (result == node.ku8MBSuccess) {
    uint32_t raw = ((uint32_t)node.getResponseBuffer(0) << 16) | node.getResponseBuffer(1);
    return *(float*)&raw;
  } else {
    Serial.printf("Eroare Modbus la adresa %u (cod %u)\n", address, result);
    return -1;
  }
}

void publishValue(const char* topic, float value, const char* label, const char* unit) {
  if (value != -1 && client.connected()) {
    char buf[10];
    dtostrf(value, 4, 2, buf);
    if (client.publish(topic, buf)) {
      Serial.printf("%s: %s %s\n", label, buf, unit);
    } else {
      Serial.printf("Eroare la trimiterea %s\n", label);
    }
  }
}

void setup() {
  Serial.begin(115200);
  Serial2.begin(9600, SERIAL_8N1, 16, 17);
  pinMode(MAX485_RE_DE, OUTPUT);
  digitalWrite(MAX485_RE_DE, LOW);

  node.begin(1, Serial2);
  node.preTransmission(preTransmission);
  node.postTransmission(postTransmission);

  setup_wifi();
  client.setServer(mqtt_server, mqtt_port);
  lastReconnectAttempt = millis();
}

void readAndPublishAll() {
  publishValue("janitza/sensor/voltageL1", readFloat(19000), "Tensiune L1", "V");
  publishValue("janitza/sensor/currentL1", readFloat(19012), "Curent L1", "A");

  float activePower = readFloat(19020);
  if (activePower != -1) activePower /= 1000.0;
  publishValue("janitza/sensor/activePowerL1", activePower, "Putere activă", "kW");

  float reactivePower = readFloat(19036);
  if (reactivePower != -1) reactivePower /= 1000.0;
  publishValue("janitza/sensor/reactivePowerL1", reactivePower, "Putere reactivă", "kvar");

  publishValue("janitza/sensor/powerFactorL1", readFloat(19044), "Factor de putere", "");
  publishValue("janitza/sensor/frequency", readFloat(19050), "Frecvență", "Hz");

  float energy = readFloat(19054);
  if (energy != -1) energy /= 1000.0;
  publishValue("janitza/sensor/energyL1", energy, "Energie L1", "kWh");
}

void loop() {
  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("WiFi căzut, reîncerc...");
    setup_wifi();
  }

  if (!client.connected()) {
    unsigned long now = millis();
    if (now - lastReconnectAttempt > reconnectInterval) {
      lastReconnectAttempt = now;
      reconnect_mqtt();
    }
  } else {
    client.loop();
    readAndPublishAll();
    delay(5000);
  }
}
