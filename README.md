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

void setup() {
    Serial.begin(115200);
    dhtSensor.setup(DHT_PIN, DHTesp::DHT22);
    Wire.begin(21, 22);
    Serial.println("Sensörler Başlatıldı...");
}

void loop() {
    // DHT22 Veri Okuma
    TempAndHumidity data = dhtSensor.getTempAndHumidity();
    Serial.print("Temp: "); Serial.print(data.temperature, 1);
    Serial.print("°C, Humidity: "); Serial.print(data.humidity, 1);
    Serial.println("%");
    
    // FS3000 Veri Okuma (I2C)
    Wire.beginTransmission(FS3000_I2C_ADDR);
    Wire.write(0x00);
    Wire.endTransmission(false);
    Wire.requestFrom(FS3000_I2C_ADDR, 2);
    float airFlowFS3000 = 0.0;
    if (Wire.available() == 2) {
        uint8_t highByte = Wire.read();
        uint8_t lowByte = Wire.read();
        uint16_t rawData = (highByte << 8) | lowByte;
        airFlowFS3000 = rawData * 0.00391;
        Serial.print("Hava Akış Hızı (FS3000): "); Serial.print(airFlowFS3000, 2); Serial.println(" m/s");
    } else {
        Serial.println("FS3000 veri okunamadı!");
    }
    
    // Analog Sensör Veri Okuma
    int sensorValue = analogRead(SENSOR_PIN);
    float voltage = ((sensorValue / (float)adcResolution) * supplyVoltage) * voltageDividerFactor;
    float flowRate = voltage * scaleFactor;
    Serial.print("ADC Value: "); Serial.print(sensorValue);
    Serial.print(" | Voltage: "); Serial.print(voltage, 3);
    Serial.print(" V | Flow Rate: "); Serial.print(flowRate, 2);
    Serial.println(" mL/s");
    
    delay(1000);
}
