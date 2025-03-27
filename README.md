#include "DHTesp.h"
#include <Wire.h>

#define DHT_PIN     15
#define FS3000_I2C_ADDR 0x28
#define SENSOR_PIN  34

const float supplyVoltage = 3.3;
const int adcResolution = 4095;
const float scaleFactor = 5.0;
const float voltageDividerFactor = 4.5 / 3.3;

DHTesp dhtSensor;

// Zamanlama için değişkenler
unsigned long previousDHTMillis = 0;
unsigned long previousFS3000Millis = 0;
unsigned long previousAnalogMillis = 0;

const unsigned long dhtInterval = 2000;
const unsigned long fs3000Interval = 1000;
const unsigned long analogInterval = 500;

// Setup
void setup() {
    Serial.begin(115200);
    dhtSensor.setup(DHT_PIN, DHTesp::DHT22);
    Wire.begin(21, 22);
    Serial.println("Sensörler Başlatıldı...");
}

// Döngü
void loop() {
    unsigned long currentMillis = millis();

    if (currentMillis - previousDHTMillis >= dhtInterval) {
        previousDHTMillis = currentMillis;
        readDHT22();
    }

    if (currentMillis - previousFS3000Millis >= fs3000Interval) {
        previousFS3000Millis = currentMillis;
        readFS3000();
    }

    if (currentMillis - previousAnalogMillis >= analogInterval) {
        previousAnalogMillis = currentMillis;
        readAnalogSensor();
    }
}

// Fonksiyon: DHT22 Sensöründen Veri Okuma
void readDHT22() {
    TempAndHumidity data = dhtSensor.getTempAndHumidity();
    if (!isnan(data.temperature) && !isnan(data.humidity)) {
        Serial.print("{\"temperature\": "); Serial.print(data.temperature, 1);
        Serial.print(", \"humidity\": "); Serial.print(data.humidity, 1);
        Serial.println("}");
    } else {
        Serial.println("Hata: DHT22 verisi okunamadı!");
    }
}

// Fonksiyon: FS3000 Hava Akış Sensöründen Veri Okuma
void readFS3000() {
    Wire.beginTransmission(FS3000_I2C_ADDR);
    Wire.write(0x00);
    if (Wire.endTransmission(false) == 0) {  // Başarılı haberleşme kontrolü
        Wire.requestFrom(FS3000_I2C_ADDR, 2);
        if (Wire.available() == 2) {
            uint8_t highByte = Wire.read();
            uint8_t lowByte = Wire.read();
            uint16_t rawData = (highByte << 8) | lowByte;
            float airFlowFS3000 = rawData * 0.00391;
            Serial.print("{\"airflow\": "); Serial.print(airFlowFS3000, 2);
            Serial.println("}");
        } else {
            Serial.println("Hata: FS3000 veri okunamadı!");
        }
    } else {
        Serial.println("Hata: FS3000 I2C bağlantısı başarısız!");
    }
}

// Fonksiyon: Analog Sensörden Veri Okuma (Ortalamalı)
void readAnalogSensor() {
    int numSamples = 10;
    int totalValue = 0;
    
    for (int i = 0; i < numSamples; i++) {
        totalValue += analogRead(SENSOR_PIN);
        delay(10);  // Gürültüyü azaltmak için küçük gecikme
    }
    
    int sensorValue = totalValue / numSamples;
    float voltage = ((sensorValue / (float)adcResolution) * supplyVoltage) * voltageDividerFactor;
    float flowRate = voltage * scaleFactor;
    
    Serial.print("{\"adc\": "); Serial.print(sensorValue);
    Serial.print(", \"voltage\": "); Serial.print(voltage, 3);
    Serial.print(", \"flowRate\": "); Serial.print(flowRate, 2);
    Serial.println("}");
}
