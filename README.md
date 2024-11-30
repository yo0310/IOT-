#include <WiFi.h>
#include <FirebaseESP32.h>
#include <Adafruit_AHTX0.h>

// Wi-Fi 및 Firebase 설정
#define FIREBASE_HOST "https://esp32devmoduie-default-rtdb.firebaseio.com/" // Firebase Realtime Database URL
#define FIREBASE_AUTH "e88WgYT9HrP6KxTFKqi16Ubu4Z5otOxyJ8KXaKtl"  // Firebase 인증 토큰
const char* ssid = "i2r";
const char* password = "00000000";

// Firebase 객체 생성
FirebaseData firebaseData;

// 센서 객체 생성
Adafruit_AHTX0 aht;

// GPIO 핀 설정
const int outputPins[] = {26, 27, 32, 33};
const int numPins = sizeof(outputPins) / sizeof(outputPins[0]);

// 타임 슬롯 데이터 구조
struct TimeSlot {
  int startHour;
  int startMinute;
  int endHour;
  int endMinute;
  String repeatMode; // "daily" or "weekly"
  int dayOfWeek;     // 0 = Sunday, ..., 6 = Saturday
};

std::vector<TimeSlot> timeSlots[numPins];

// NTP 설정 (시간 동기화)
const char* ntpServer = "pool.ntp.org";
const long gmtOffset_sec = 3600 * 9; // KST (UTC+9)
const int daylightOffset_sec = 0;

// 함수 선언
void setupWiFi();
void syncTime();
void readSensorData();
void loadTimeSlots();
void updatePinStates();

void setup() {
  Serial.begin(115200);

  // Wi-Fi 연결
  setupWiFi();

  // Firebase 초기화
  Firebase.begin(FIREBASE_HOST, FIREBASE_AUTH);
  Firebase.reconnectWiFi(true);

  // 센서 초기화
  if (!aht.begin()) {
    Serial.println("AHT sensor not found!");
    while (1) delay(10);
  }

  // GPIO 핀 초기화
  for (int i = 0; i < numPins; i++) {
    pinMode(outputPins[i], OUTPUT);
    digitalWrite(outputPins[i], LOW);
  }

  // 시간 동기화
  syncTime();

  // Firebase에서 타임 슬롯 로드
  loadTimeSlots();
}

void loop() {
  // 센서 데이터 읽기 및 Firebase 업데이트
  readSensorData();

  // 핀 상태 업데이트
  updatePinStates();

  // Firebase 명령 처리
  if (Firebase.getJSON(firebaseData, "/commands")) {
    if (firebaseData.dataType() == "json") {
      DynamicJsonDocument doc(1024);
      deserializeJson(doc, firebaseData.jsonData());
      for (JsonPair cmd : doc.as<JsonObject>()) {
        int pinIndex = cmd.value()["pin"];
        bool state = cmd.value()["state"];
        if (pinIndex >= 0 && pinIndex < numPins) {
          digitalWrite(outputPins[pinIndex], state ? HIGH : LOW);
          Serial.printf("Pin %d set to %s\n", outputPins[pinIndex], state ? "HIGH" : "LOW");
        }
      }
    }
    Firebase.deleteNode(firebaseData, "/commands"); // 명령 삭제
  }

  delay(1000);
}

void setupWiFi() {
  Serial.print("Connecting to Wi-Fi");
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.print(".");
  }
  Serial.println("\nWi-Fi connected");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());
}

void syncTime() {
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  struct tm timeInfo;
  if (!getLocalTime(&timeInfo)) {
    Serial.println("Failed to obtain time");
    return;
  }
  Serial.println(&timeInfo, "%Y-%m-%d %H:%M:%S");
}

void readSensorData() {
  sensors_event_t humidity, temp;
  aht.getEvent(&humidity, &temp);

  float temperature = temp.temperature;
  float humidityVal = humidity.relative_humidity;

  Serial.printf("Temp: %.1f°C, Humi: %.1f%%\n", temperature, humidityVal);

  Firebase.setFloat(firebaseData, "/sensors/temp", temperature);
  Firebase.setFloat(firebaseData, "/sensors/humi", humidityVal);
}

void loadTimeSlots() {
  if (Firebase.getJSON(firebaseData, "/timeSlots")) {
    if (firebaseData.dataType() == "json") {
      DynamicJsonDocument doc(2048);
      deserializeJson(doc, firebaseData.jsonData());
      for (JsonPair slot : doc.as<JsonObject>()) {
        int pin = slot.value()["pin"];
        if (pin >= 0 && pin < numPins) {
          TimeSlot ts;
          ts.startHour = slot.value()["startHour"];
          ts.startMinute = slot.value()["startMinute"];
          ts.endHour = slot.value()["endHour"];
          ts.endMinute = slot.value()["endMinute"];
          ts.repeatMode = slot.value()["repeatMode"].as<String>();
          ts.dayOfWeek = slot.value()["dayOfWeek"];
          timeSlots[pin].push_back(ts);
        }
      }
    }
  }
}

void updatePinStates() {
  struct tm timeInfo;
  if (!getLocalTime(&timeInfo)) return;

  int currentHour = timeInfo.tm_hour;
  int currentMinute = timeInfo.tm_min;
  int currentDayOfWeek = timeInfo.tm_wday;

  for (int i = 0; i < numPins; i++) {
    bool pinState = false;
    for (TimeSlot ts : timeSlots[i]) {
      if (ts.repeatMode == "daily" ||
          (ts.repeatMode == "weekly" && ts.dayOfWeek == currentDayOfWeek)) {
        int startMinutes = ts.startHour * 60 + ts.startMinute;
        int endMinutes = ts.endHour * 60 + ts.endMinute;
        int currentMinutes = currentHour * 60 + currentMinute;

        if (currentMinutes >= startMinutes && currentMinutes < endMinutes) {
          pinState = true;
          break;
        }
      }
    }
    digitalWrite(outputPins[i], pinState ? HIGH : LOW);
  }
}

#아두이노 프로그램 입니다.
