#include <DHT.h>

//  Klima Üfleme Sensörü (FS300A veya analog giriş)
#define FAN_SENSOR_PIN A0  
int fanThreshold = 300;  // Üfleme algılama eşiği

//  Sıcaklık & Nem Sensörü (DHT22)
#define DHTPIN 2       
#define DHTTYPE DHT22  
DHT dht(DHTPIN, DHTTYPE);

//  Sıcaklık ve Nem Alarm Eşik Değerleri
#define MIN_TEMP 18
#define MAX_TEMP 24
#define MIN_HUMIDITY 45
#define MAX_HUMIDITY 65

//  Hall Effect Sensörü (Elektriksel Akım Tespiti)
#define HALL_SENSOR_PIN A1   // Hall Effect sensörünün bağlı olduğu pin
int hallThreshold = 300;  // Hall sensöründen alınan değeri değerlendirme eşiği

void setup() {
  Serial.begin(115200);
  pinMode(FAN_SENSOR_PIN, INPUT);
  pinMode(HALL_SENSOR_PIN, INPUT);  // Hall Effect sensörünü giriş olarak ayarla
  dht.begin();
}

void loop() {
  checkFan();  // Klima üflemesini kontrol et
  checkTemperatureHumidity();  //  Sıcaklık & Nem kontrol et
  checkHallSensor();  //  Hall Effect sensörünü kontrol et
  delay(3000);  // 3 saniyede bir kontrol
}

//  Klima Üfleme Kontrolü
void checkFan() {
  int fanValue = analogRead(FAN_SENSOR_PIN);
  Serial.print("Klima Üfleme Sensörü Değeri: ");
  Serial.println(fanValue);

  if (fanValue < fanThreshold) {  
    Serial.println(" UYARI! Klima üflemeyi durdurdu! ");
  } else {
    Serial.println("Klima çalışıyor.");
  }
}

//  Sıcaklık ve Nem Kontrolü
void checkTemperatureHumidity() {
  float temperature = dht.readTemperature();
  float humidity = dht.readHumidity();

  Serial.print("Sıcaklık: "); 
  Serial.print(temperature);
  Serial.print(" °C  |  Nem: "); 
  Serial.print(humidity);
  Serial.println(" %");

  if (temperature < MIN_TEMP || temperature > MAX_TEMP) {
    Serial.println(" UYARI! Sıcaklık güvenli aralığın dışında! ");
  }

  if (humidity < MIN_HUMIDITY || humidity > MAX_HUMIDITY) {
    Serial.println(" UYARI! Nem güvenli aralığın dışında! ");
  }
}

//  Hall Effect Sensörü Kontrolü (Elektriksel Akım Tespiti)
void checkHallSensor() {
  int hallValue = analogRead(HALL_SENSOR_PIN);  // Hall Effect sensöründen veri oku
  Serial.print("Hall Effect Sensörü Değeri: ");
  Serial.println(hallValue);

  if (hallValue < hallThreshold) {  // Eşik değerin altındaysa, elektrik yok ya da düşük akım
    Serial.println(" UYARI! Elektriksel akımda bir sorun tespit edildi! ");
  } else {
    Serial.println(" Elektriksel akım normal.");
  }
}
