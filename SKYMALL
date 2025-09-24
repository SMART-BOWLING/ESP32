// скетч автоматически переводит на летнее и зимнее время в Украине

#include <WiFiClientSecure.h>
#include <HTTPClient.h>
#include <WiFi.h>
#include <GyverButton.h>
#include "time.h"

// ===================== НАСТРОЙКИ =====================
#define SENSOR_PIN 4      // Пин датчика
#define BUTTON_PIN 16     // Пин кнопки

// ===================== ПАРАМЕТРЫ ДЛЯ ДАТЧИКА =====================
String sensorSheetName = "PS1";

// ===================== ПАРАМЕТРЫ ДЛЯ КНОПКИ =====================
String buttonColumn = "F";

// ============== URLs для Google Apps Script ==============
const char* sensorGoogleScriptURL = "https://script.google.com/macros/s/AKfycbw8kYOi_3J24EecI9GRXmlo0EbY7sxdE0yJwTQHk04Qc1ru16R4bNJ1oXmT-THPpVe2/exec";
const char* buttonGoogleScriptURL = "https://script.google.com/macros/s/AKfycbxwStfnf_Ts2EZ8S7llITqvwOWF5yVPcq5rt5eYIv05KNN7z55cCTYblBAuevmrxUpS/exec";

// ===================== Wi-Fi настройки =====================
#define WIFI_SSID "bowling service"
#define WIFI_PASS "ls77777779"
const unsigned long debounceDelaySensor = 50;
const unsigned long wifiTimeout = 30000;

// ===================== ПИНЫ СВЕТОДИОДОВ =====================
const int ledRedPin = 5;
const int ledGreenPin = 17;

// ===================== ПЕРЕМЕННЫЕ =====================
GButton button(BUTTON_PIN);
unsigned long lastDebounceTimeSensor = 0;
unsigned long lastMinute = 0;
int sensorState = HIGH;
int sensorPrevState = HIGH;
int sensorTriggerCount = 0;
bool eventActive = false;
unsigned long eventStartTime = 0;
#define EVENT_DURATION 3000

unsigned long lastBlinkTime = 0;
const unsigned long blinkInterval = 500;
bool blinkState = false;

// ===================== SETUP =====================
void setup() {
  Serial.begin(115200);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  Serial.println("Подключение к Wi-Fi...");
  button.setTickMode(AUTO);
  button.setTimeout(3000);
  pinMode(SENSOR_PIN, INPUT_PULLUP);
  sensorPrevState = digitalRead(SENSOR_PIN);
  pinMode(ledRedPin, OUTPUT);
  pinMode(ledGreenPin, OUTPUT);
  digitalWrite(ledRedPin, LOW);
  digitalWrite(ledGreenPin, LOW);

  // Настройка NTP
  #define TZ_EUROPE_KYIV "EET-2EEST,M3.5.0/3,M10.5.0/4"
  configTzTime(TZ_EUROPE_KYIV, "pool.ntp.org", "time.nist.gov");
  Serial.println("Ожидание синхронизации времени...");
  while (time(nullptr) < 100000) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nВремя синхронизировано");
  printCurrentTimezone();

  struct tm timeinfo;
  getLocalTime(&timeinfo);
  lastMinute = timeinfo.tm_min;
}

// ===================== LOOP =====================
void loop() {
  button.tick();
  checkSensor();
  checkButton();
  checkWiFiConnection();
  handleLedIndication();
  handleMinuteChange();
}

// ===================== ОБРАБОТКА КНОПКИ =====================
void checkButton() {
  if (button.isClick()) {
    sendButtonData();
    digitalWrite(ledRedPin, HIGH);
    digitalWrite(ledGreenPin, LOW);
    delay(3000);
    restoreWiFiLedState();
  }
  if (button.isHolded()) {
    Serial.println("Перезагрузка ESP32...");
    ESP.restart();
  }
}

// ===================== ОБРАБОТКА ДАТЧИКА =====================
void checkSensor() {
  int sensorReading = digitalRead(SENSOR_PIN);
  if (sensorReading != sensorPrevState) {
    lastDebounceTimeSensor = millis();
    sensorPrevState = sensorReading;
  }
  if ((millis() - lastDebounceTimeSensor > debounceDelaySensor) && (sensorReading != sensorState)) {
    sensorState = sensorReading;
    if (sensorState == LOW) {
      sensorTriggerCount++;
      Serial.println("Sensor triggered, count=" + String(sensorTriggerCount));
      activateRedEvent();
    }
  }
}

// ===================== ОБРАБОТКА И ОТПРАВКА КАЖДУЮ МИНУТУ =====================
void handleMinuteChange() {
  struct tm timeinfo;
  getLocalTime(&timeinfo);
  if (timeinfo.tm_min != lastMinute) {
    sendSensorBatchData(sensorTriggerCount, timeinfo);
    sensorTriggerCount = 0;
    lastMinute = timeinfo.tm_min;
  }
}

// ===================== ОТПРАВКА ДАННЫХ ДАТЧИКА =====================
void sendSensorBatchData(int count, struct tm timeinfo) {
  timeinfo.tm_min--;
  if (timeinfo.tm_min < 0) {
    timeinfo.tm_min = 59;
    timeinfo.tm_hour--;
  }
  if (timeinfo.tm_hour < 0) {
    timeinfo.tm_hour = 23;
    timeinfo.tm_mday--;
  }
  char timeStr[20];
  strftime(timeStr, sizeof(timeStr), "%Y-%m-%d %H:%M:00", &timeinfo);
  String url = String(sensorGoogleScriptURL) + "?sheet=" + urlencode(sensorSheetName) + "&count=" + String(count) + "&time=" + urlencode(String(timeStr));
  WiFiClientSecure client;
  client.setInsecure();
  HTTPClient http;
  if (WiFi.status() == WL_CONNECTED) {
    http.begin(client, url);
    int httpCode = http.GET();
    Serial.printf("SensorBatch sent OK, code: %d, time: %s\n", httpCode, timeStr);
    http.end();
  }
}

// ===================== ОТПРАВКА ДАННЫХ КНОПКИ =====================
void sendButtonData() {
  String url = String(buttonGoogleScriptURL) + "?column=" + urlencode(buttonColumn);
  WiFiClientSecure client;
  client.setInsecure();
  HTTPClient http;
  if (WiFi.status() == WL_CONNECTED) {
    http.begin(client, url);
    int httpCode = http.GET();
    Serial.printf("ButtonData sent OK, code: %d\n", httpCode);
    http.end();
  }
}

// ===================== ПРОВЕРКА Wi-Fi =====================
void checkWiFiConnection() {
  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("WiFi не подключен");
  }
}

// ===================== СВЕТОДИОДЫ =====================
void handleLedIndication() {
  if (eventActive) {
    if (millis() - eventStartTime < EVENT_DURATION) {
      digitalWrite(ledRedPin, HIGH);
      digitalWrite(ledGreenPin, LOW);
    } else {
      eventActive = false;
      restoreWiFiLedState();
    }
    return;
  }
  if (WiFi.status() != WL_CONNECTED) {
    if (millis() - lastBlinkTime >= blinkInterval) {
      lastBlinkTime = millis();
      blinkState = !blinkState;
      digitalWrite(ledRedPin, blinkState ? HIGH : LOW);
      digitalWrite(ledGreenPin, LOW);
    }
  } else {
    digitalWrite(ledGreenPin, HIGH);
    digitalWrite(ledRedPin, LOW);
  }
}

void activateRedEvent() {
  eventActive = true;
  eventStartTime = millis();
  digitalWrite(ledRedPin, HIGH);
  digitalWrite(ledGreenPin, LOW);
}

void restoreWiFiLedState() {
  if (WiFi.status() == WL_CONNECTED) {
    digitalWrite(ledGreenPin, HIGH);
    digitalWrite(ledRedPin, LOW);
  } else {
    digitalWrite(ledGreenPin, LOW);
    digitalWrite(ledRedPin, HIGH);
  }
}

// ===================== ВЫВОД ЧАСОВОГО ПОЯСА =====================
void printCurrentTimezone() {
  struct tm timeinfo;
  getLocalTime(&timeinfo);
  char tzString[10];
  strftime(tzString, sizeof(tzString), "%Z", &timeinfo);
  Serial.printf("Текущий часовой пояс: %s\n", tzString);
}

// ===================== URLENCODE =====================
String urlencode(const String &str) {
  String encoded = "";
  char c;
  char code0;
  char code1;
  for (int i = 0; i < str.length(); i++) {
    c = str.charAt(i);
    if (isalnum(c)) {
      encoded += c;
    } else {
      code1 = (c & 0xf) + '0';
      if ((c & 0xf) > 9) code1 = (c & 0xf) - 10 + 'A';
      code0 = ((c >> 4) & 0xf) + '0';
      if (((c >> 4) & 0xf) > 9) code0 = ((c >> 4) & 0xf) - 10 + 'A';
      encoded += '%';
      encoded += code0;
      encoded += code1;
    }
  }
  return encoded;
}
