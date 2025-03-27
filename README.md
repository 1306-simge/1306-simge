#include <ESP8266WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
#include "DHTesp.h"
#include <Wire.h>

// Wi-Fi ve MQTT Ayarları
const char* ssid = "WiFi_ADINIZ";
const char* password = "WiFi_SIFRENIZ";
const char* mqtt_server = "broker.hivemq.com";
const char* mqtt_topic = "BilSensor/Data"; 

WiFiClient espClient;
PubSubClient client(espClient);

// Sensör Pinleri
#define DHT_PIN  D4  // DHT22 sıcaklık & nem
#define SENSOR_PIN A0 // Analog sensör
#define FS3000_I2C_ADDR 0x28 // FS3000 hava akış sensörü adresi

DHTesp dhtSensor;

// Zamanlama için değişken
unsigned long previousMillis = 0;
const unsigned long interval = 2000;  // 2 saniyede bir veri gönder

void setupWiFi() {
    Serial.print("Wi-Fi bağlanıyor...");
    WiFi.begin(ssid, password);
    while (WiFi.status() != WL_CONNECTED) {
        delay(500);
        Serial.print(".");
    }
    Serial.println(" Bağlandı!");
}

void reconnect() {
    while (!client.connected()) {
        Serial.print("MQTT bağlanıyor...");
        if (client.connect("ESP8266_Client")) {
            Serial.println(" Bağlandı!");
        } else {
            Serial.print("Bağlantı hatası: ");
            Serial.println(client.state());
            delay(5000);
        }
    }
}

void setup() {
    Serial.begin(115200);
    dhtSensor.setup(DHT_PIN, DHTesp::DHT22);
    Wire.begin(D2, D1); // I2C için SDA ve SCL pinleri

    setupWiFi();
    client.setServer(mqtt_server, 1883);
}

void loop() {
    if (!client.connected()) {
        reconnect();
    }
    client.loop();

    unsigned long currentMillis = millis();
    if (currentMillis - previousMillis >= interval) {
        previousMillis = currentMillis;
        sendSensorData();
    }
}

void sendSensorData() {
    // DHT22 verisi al
    TempAndHumidity data = dhtSensor.getTempAndHumidity();
    float temperature = data.temperature;
    float humidity = data.humidity;

    // FS3000 hava akış verisi al
    Wire.beginTransmission(FS3000_I2C_ADDR);
    Wire.write(0x00);
    Wire.endTransmission(false);
    Wire.requestFrom(FS3000_I2C_ADDR, 2);
    float airFlow = 0.0;
    if (Wire.available() == 2) {
        uint8_t highByte = Wire.read();
        uint8_t lowByte = Wire.read();
        uint16_t rawData = (highByte << 8) | lowByte;
        airFlow = rawData * 0.00391;
    }

    // Analog sensörden veri oku
    int sensorValue = analogRead(SENSOR_PIN);
    float voltage = (sensorValue / 1023.0) * 3.3;
    float flowRate = voltage * 5.0;

    // JSON formatında veriyi oluştur
    StaticJsonDocument<200> jsonDoc;
    jsonDoc["ID"] = "BilSensor";
    jsonDoc["Temp"] = temperature;
    jsonDoc["Hum"] = humidity;
    jsonDoc["Flow"] = flowRate;
    
    char payload[200];
    serializeJson(jsonDoc, payload);

    Serial.println(payload);
    client.publish(mqtt_topic, payload);
}

