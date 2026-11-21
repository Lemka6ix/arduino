# arduino
## 10

```C
#include <M5Unified.h>
#include <math.h>

bool alarmActive = false;

float base_ax = 0, base_ay = 0, base_az = 0;
float base_norm = 1.0;

const float ANGLE_THRESHOLD = 7.0;  
const float GYRO_THRESHOLD  = 1.2;  

// Переменные для звука
unsigned long previousToneMillis = 0;
int toneState = 0;
const int TONE_DURATION = 150; 
const int PAUSE_DURATION = 50; 

void calibrate() {
  M5.Imu.update();
  auto d = M5.Imu.getImuData();
  base_ax = d.accel.x;
  base_ay = d.accel.y;
  base_az = d.accel.z;
  base_norm = sqrt(base_ax * base_ax + base_ay * base_ay + base_az * base_az);
  if (base_norm < 0.1) base_norm = 0.1;
}

void drawIdleScreen() {
  M5.Lcd.fillScreen(TFT_BLACK);
  M5.Lcd.setTextColor(TFT_WHITE);
  M5.Lcd.setTextSize(2);
  M5.Lcd.setCursor(10, 20);
  M5.Lcd.println("ALARM SYSTEM");
  M5.Lcd.setTextSize(1);
  M5.Lcd.setCursor(10, 60);
  M5.Lcd.println("Status: DISARMED");
  M5.Lcd.println("BtnA to reset");
}

void setup() {
  M5.begin();
  M5.Lcd.setRotation(0);
  drawIdleScreen();
  calibrate();
  delay(1000);
  M5.Lcd.fillRect(0, 40, 135, 30, TFT_BLACK);
}

void loop() {
  M5.update();

  if (M5.BtnA.wasPressed()) {
    if (alarmActive) {
      alarmActive = false;
      M5.Speaker.stop();          
      toneState = 0;              
      drawIdleScreen();
      calibrate();
    }
  }

  if (M5.Imu.update()) {
    auto d = M5.Imu.getImuData();

    float ax = d.accel.x;
    float ay = d.accel.y;
    float az = d.accel.z;
    float norm = sqrt(ax * ax + ay * ay + az * az);
    if (norm < 0.1) norm = 0.1;

    float dot = ax * base_ax + ay * base_ay + az * base_az;
    float cosAng = dot / (norm * base_norm);
    if (cosAng >  1.0) cosAng =  1.0;
    if (cosAng < -1.0) cosAng = -1.0;
    float angleDeg = acos(cosAng) * 180.0 / M_PI;

    float gmag = sqrt(d.gyro.x * d.gyro.x +
                      d.gyro.y * d.gyro.y +
                      d.gyro.z * d.gyro.z);

    if (angleDeg > ANGLE_THRESHOLD || gmag > GYRO_THRESHOLD) {
      alarmActive = true;
    }
  }

  if (alarmActive) {
    // Мигание экрана
    static unsigned long lastBlink = 0;
    if (millis() - lastBlink > 200) {
      lastBlink = millis();
      static bool isRed = false;
      isRed = !isRed;
      M5.Lcd.fillScreen(isRed ? TFT_RED : TFT_BLACK);
      M5.Lcd.setTextColor(isRed ? TFT_WHITE : TFT_RED);
      M5.Lcd.setCursor(20, 20);
      M5.Lcd.setTextSize(2);
      M5.Lcd.println("ALARM!");
      M5.Lcd.setCursor(10, 100);
      M5.Lcd.setTextSize(1);
      M5.Lcd.println("Press BtnA to stop");
    }

    // Двухтональный звук
    if (millis() - previousToneMillis >= (TONE_DURATION + PAUSE_DURATION)) {
      previousToneMillis = millis();
      if (toneState == 0) {
        M5.Speaker.tone(1000, TONE_DURATION);
        toneState = 1;
      } else {
        M5.Speaker.tone(2000, TONE_DURATION);
        toneState = 0;
      }
    }
  } else {
    delay(500);
  }
}
```


## 12
```C
#include <M5Unified.h>
#include <math.h>

const int BUFFER_SIZE = 512;
int16_t buffer[BUFFER_SIZE];
const float CLAP_THRESHOLD = 2000.0;

uint16_t colors[] = {TFT_RED, TFT_BLUE, TFT_GREEN, TFT_YELLOW, TFT_MAGENTA, TFT_CYAN};
int colorIndex = 0;

void setup() {
  M5.begin();
  M5.Display.setRotation(0);
  M5.Display.fillScreen(TFT_BLACK);
  
  M5.Mic.begin();
  M5.Mic.setSampleRate(8000);
}

void loop() {
  M5.update();
  
  bool ok = M5.Mic.record(buffer, BUFFER_SIZE);
  if (ok) {
    while (M5.Mic.isRecording()) {
      delay(1);
    }
    
    double sum = 0;
    for (int i = 0; i < BUFFER_SIZE; i++) {
      sum += (double)buffer[i] * (double)buffer[i];
    }
    double rms = sqrt(sum / BUFFER_SIZE);
    
    if (rms > CLAP_THRESHOLD) {
      colorIndex++;
      if (colorIndex >= 6) {
        colorIndex = 0;
      }
      M5.Display.fillScreen(colors[colorIndex]);
      delay(300); 
    }
  }
}
```


## 13
```C
#include <M5Unified.h>
#include "USB.h"
#include "USBHIDKeyboard.h"
#include "USBHIDMouse.h"

USBHIDKeyboard Keyboard;
USBHIDMouse Mouse;

const char* MY_NAME = "Klim"; 

bool executed = false;

void pressCombo(uint8_t mod, uint8_t key, uint16_t hold = 60) {
  if (mod) Keyboard.press(mod);
  Keyboard.press(key);
  delay(hold);
  Keyboard.releaseAll();
  delay(150);
}

void typeSlow(const char* s) {
  while (*s) {
    if (*s == '\n') {
      Keyboard.press(KEY_RETURN);
      delay(30);
      Keyboard.releaseAll();
    } else {
      Keyboard.print(*s);
    }
    delay(25);
    s++;
  }
}

// Открыть Блокнот через Win+R
void openNotepad() {
  pressCombo(KEY_LEFT_GUI, 'r');     
  delay(500);
  typeSlow("notepad");              
  delay(150);
  Keyboard.press(KEY_RETURN);
  delay(30);
  Keyboard.releaseAll();
  delay(2200);                      
}

// Сохранить как файл
void saveAs(const char* filename) {
  pressCombo(KEY_LEFT_CTRL, 's');    // Ctrl+S
  delay(700);
  typeSlow(filename);
  delay(300);
  Keyboard.press(KEY_RETURN);
  delay(30);
  Keyboard.releaseAll();
  delay(1200);
}

void runPoem() {
  M5.Display.fillScreen(TFT_BLACK);
  M5.Display.setTextColor(TFT_WHITE, TFT_BLACK);
  M5.Display.setTextSize(1);

  M5.Display.drawString("1. Open Notepad...", 10, 10);
  openNotepad();

  M5.Display.drawString("2. Type poem...", 10, 30);
  typeSlow(
    "Roses are red,\n"
    "Violets are blue,\n"
    "Sugar is sweet,\n"
    "And so are you.\n"
  );
  delay(500);

  M5.Display.drawString("3. Save file...", 10, 50);
  String fname = String(MY_NAME) + ".txt";
  saveAs(fname.c_str());

  M5.Display.setTextColor(TFT_GREEN, TFT_BLACK);
  M5.Display.drawString("Done!", 10, 70);
}

void setup() {
  M5.begin();
  M5.Display.setRotation(0); 
  M5.Display.fillScreen(TFT_BLACK);
  M5.Display.setTextColor(TFT_WHITE, TFT_BLACK);
  M5.Display.setTextSize(1);
  M5.Display.drawString("HID Keyboard ready", 10, 10);
  M5.Display.drawString("Press A to run", 10, 30);
  M5.Display.drawString("OS layout: ENGLISH", 10, 50);

  Keyboard.begin();
  Mouse.begin();
  USB.begin();
  delay(1500);                        
}

void loop() {
  M5.update();
  if (M5.BtnA.wasPressed() && !executed) {
    executed = true;
    runPoem();
  }
  delay(10);
}
```

## 15

```C
#include <M5Unified.h>
#include <WiFi.h>
#include <PubSubClient.h>

const char* ssid = "ASOIU";
const char* password = "kaf.asoiu.48";

const char* mqtt_server = "broker.hivemq.com";
const int mqtt_port = 1883;

String clientId = "M5StickS3-Klim-"; 
String topic_btn = "M5_Klim/buttons";
String topic_text = "M5_Klim/text";
String topic_bool = "M5_Klim/bool";

WiFiClient espClient;
PubSubClient client(espClient);

String currentDisplayText = "No text yet";
bool updateDisplay = false;
bool playSoundFlag = false;

void callback(char* topic, byte* payload, unsigned int length) {
  String message = "";
  for (int i = 0; i < length; i++) {
    message += (char)payload[i];
  }

  String strTopic = String(topic);

  if (strTopic == topic_text) {
    currentDisplayText = message;
    updateDisplay = true;
  } 
  else if (strTopic == topic_bool) {
    if (message == "1" || message == "true" || message == "TRUE") {
      playSoundFlag = true;
    }
  }
}

void reconnect() {
  while (!client.connected()) {
    M5.Display.fillRect(0, 20, 135, 20, TFT_BLACK);
    M5.Display.setCursor(10, 20);
    M5.Display.setTextColor(TFT_YELLOW);
    M5.Display.println("Connecting MQTT...");
    
    clientId = "M5StickS3-" + String(random(0xffff), HEX);
    
    if (client.connect(clientId.c_str())) {
      M5.Display.fillRect(0, 20, 135, 20, TFT_BLACK);
      M5.Display.setCursor(10, 20);
      M5.Display.setTextColor(TFT_GREEN);
      M5.Display.println("MQTT Connected");
      
      client.subscribe(topic_text.c_str());
      client.subscribe(topic_bool.c_str());
    } else {
      delay(2000);
    }
  }
}

void setup() {
  M5.begin();
  M5.Display.setRotation(0);
  M5.Display.fillScreen(TFT_BLACK);
  M5.Display.setTextColor(TFT_WHITE);
  M5.Display.setTextSize(1);
  M5.Display.setCursor(10, 10);
  M5.Display.println("IoT MQTT Device");

  WiFi.begin(ssid, password);
  M5.Display.setCursor(10, 40);
  M5.Display.println("Connecting WiFi...");
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    M5.Display.print(".");
  }

  M5.Display.fillRect(0, 40, 135, 20, TFT_BLACK);
  M5.Display.setCursor(10, 40);
  M5.Display.setTextColor(TFT_GREEN);
  M5.Display.println("WiFi Connected");

  client.setServer(mqtt_server, mqtt_port);
  client.setCallback(callback);
}

void loop() {
  M5.update();

  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  if (M5.BtnA.wasPressed()) {
    client.publish(topic_btn.c_str(), "BtnA_Pressed");
  }
  if (M5.BtnB.wasPressed()) {
    client.publish(topic_btn.c_str(), "BtnB_Pressed");
  }

  if (updateDisplay) {
    updateDisplay = false;
    M5.Display.fillRect(0, 70, 135, 80, TFT_BLACK);
    M5.Display.setCursor(10, 70);
    M5.Display.setTextColor(TFT_CYAN);
    M5.Display.println("MQTT Text:");
    M5.Display.setCursor(10, 90);
    M5.Display.setTextColor(TFT_WHITE);
    M5.Display.println(currentDisplayText);
  }

  if (playSoundFlag) {
    playSoundFlag = false;
    M5.Speaker.tone(4000, 500);
    delay(600);
    M5.Speaker.tone(2000, 300);
  }

  delay(10);
}
```


## 14

```C
#include <M5Unified.h>
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>

#define SERVICE_UUID           ""
#define CHARACTERISTIC_UUID_TX ""
#define CHARACTERISTIC_UUID_RX ""

BLEServer* pServer = nullptr;
BLECharacteristic* pTxCharacteristic = nullptr;
BLECharacteristic* pRxCharacteristic = nullptr;

bool deviceConnected = false;
bool newDataReceived = false;
std::string receivedText = "";

class MyServerCallbacks : public BLEServerCallbacks {
  void onConnect(BLEServer* pServer) {
    deviceConnected = true;
  }
  void onDisconnect(BLEServer* pServer) {
    deviceConnected = false;
    BLEDevice::startAdvertising();
  }
};

class MyCallbacks : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic* pCharacteristic) {
    std::string value = pCharacteristic->getValue();
    if (value.length() > 0) {
      receivedText = value;
      newDataReceived = true;
    }
  }
};

void setup() {
  M5.begin();
  M5.Display.setRotation(0);
  M5.Display.fillScreen(TFT_BLACK);
  M5.Display.setTextColor(TFT_WHITE);
  M5.Display.setTextSize(1);
  M5.Display.setCursor(10, 10);
  M5.Display.println("BLE IMU & Text");
  M5.Display.setCursor(10, 40);
  M5.Display.println("Status: Waiting...");

  BLEDevice::init("M5StickS3_IMU");
  pServer = BLEDevice::createServer();
  pServer->setCallbacks(new MyServerCallbacks());

  BLEService* pService = pServer->createService(SERVICE_UUID);

  pTxCharacteristic = pService->createCharacteristic(
      CHARACTERISTIC_UUID_TX,
      BLECharacteristic::PROPERTY_READ | BLECharacteristic::PROPERTY_NOTIFY
  );
  pTxCharacteristic->addDescriptor(new BLE2902());

  pRxCharacteristic = pService->createCharacteristic(
      CHARACTERISTIC_UUID_RX,
      BLECharacteristic::PROPERTY_READ | BLECharacteristic::PROPERTY_WRITE
  );
  pRxCharacteristic->setCallbacks(new MyCallbacks());
  pRxCharacteristic->setValue("Hello");

  pService->start();

  BLEAdvertising* pAdvertising = BLEDevice::getAdvertising();
  pAdvertising->addServiceUUID(SERVICE_UUID);
  pAdvertising->setScanResponse(true);
  pAdvertising->setMinPreferred(0x06);
  pAdvertising->setMinPreferred(0x12);
  BLEDevice::startAdvertising();
}

void loop() {
  M5.update();

  if (deviceConnected) {
    static unsigned long lastTime = 0;
    if (millis() - lastTime >= 1000) {
      lastTime = millis();
      if (M5.Imu.update()) {
        auto data = M5.Imu.getImuData();
        char buf[64];
        snprintf(buf, sizeof(buf), "X:%.2f Y:%.2f Z:%.2f", 
                 data.accel.x, data.accel.y, data.accel.z);
        pTxCharacteristic->setValue(buf);
        pTxCharacteristic->notify();
      }
    }
  }

  if (newDataReceived) {
    newDataReceived = false;
    M5.Display.fillRect(0, 40, 135, 100, TFT_BLACK);
    M5.Display.setCursor(10, 40);
    M5.Display.setTextColor(TFT_GREEN);
    M5.Display.println("Text from BLE:");
    M5.Display.setCursor(10, 60);
    M5.Display.setTextColor(TFT_WHITE);
    M5.Display.println(receivedText.c_str());
  }

  static bool lastConnectionState = false;
  if (deviceConnected != lastConnectionState) {
    lastConnectionState = deviceConnected;
    M5.Display.fillRect(0, 20, 135, 20, TFT_BLACK);
    M5.Display.setCursor(10, 20);
    if (deviceConnected) {
      M5.Display.setTextColor(TFT_GREEN);
      M5.Display.println("Status: Connected");
    } else {
      M5.Display.setTextColor(TFT_RED);
      M5.Display.println("Status: Waiting...");
    }
  }

  delay(10);
}
```

```C
#include <M5Unified.h>
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>

#define SERVICE_UUID           "4fafc201-1fb5-459e-8fcc-c5c9c331914b"
#define CHARACTERISTIC_UUID_TX "beb5483e-36e1-4688-b7f5-ea07361b26a8"
#define CHARACTERISTIC_UUID_RX "1c95d5e3-1c31-4d3d-985c-6a52b388d1f6"

BLEServer* pServer = nullptr;
BLECharacteristic* pTxCharacteristic = nullptr;
BLECharacteristic* pRxCharacteristic = nullptr;

bool deviceConnected = false;
bool newDataReceived = false;
String receivedText = "";          // ← изменено на String

class MyServerCallbacks : public BLEServerCallbacks {
  void onConnect(BLEServer* pServer) { deviceConnected = true; }
  void onDisconnect(BLEServer* pServer) {
    deviceConnected = false;
    BLEDevice::startAdvertising();
  }
};

class MyCallbacks : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic* pCharacteristic) {
    String value = pCharacteristic->getValue();   // ← изменено на String
    if (value.length() > 0) {
      receivedText = value;
      newDataReceived = true;
    }
  }
};

void setup() {
  M5.begin();
  M5.Display.setRotation(0);
  M5.Display.fillScreen(TFT_BLACK);
  M5.Display.setTextColor(TFT_WHITE);
  M5.Display.setTextSize(1);
  M5.Display.setCursor(10, 10);
  M5.Display.println("BLE IMU & Text");
  M5.Display.setCursor(10, 40);
  M5.Display.println("Status: Waiting...");

  BLEDevice::init("M5StickS3_IMU");
  pServer = BLEDevice::createServer();
  pServer->setCallbacks(new MyServerCallbacks());

  BLEService* pService = pServer->createService(SERVICE_UUID);

  pTxCharacteristic = pService->createCharacteristic(
      CHARACTERISTIC_UUID_TX,
      BLECharacteristic::PROPERTY_READ | BLECharacteristic::PROPERTY_NOTIFY
  );
  pTxCharacteristic->addDescriptor(new BLE2902());

  pRxCharacteristic = pService->createCharacteristic(
      CHARACTERISTIC_UUID_RX,
      BLECharacteristic::PROPERTY_READ | BLECharacteristic::PROPERTY_WRITE
  );
  pRxCharacteristic->setCallbacks(new MyCallbacks());
  pRxCharacteristic->setValue("Hello");

  pService->start();

  BLEAdvertising* pAdvertising = BLEDevice::getAdvertising();
  pAdvertising->addServiceUUID(SERVICE_UUID);
  pAdvertising->setScanResponse(true);
  pAdvertising->setMinPreferred(0x06);
  pAdvertising->setMinPreferred(0x12);
  BLEDevice::startAdvertising();
}

void loop() {
  M5.update();

  if (deviceConnected) {
    static unsigned long lastTime = 0;
    if (millis() - lastTime >= 1000) {
      lastTime = millis();
      if (M5.Imu.update()) {
        auto data = M5.Imu.getImuData();
        char buf[64];
        snprintf(buf, sizeof(buf), "X:%.2f Y:%.2f Z:%.2f",
                 data.accel.x, data.accel.y, data.accel.z);
        pTxCharacteristic->setValue(buf);
        pTxCharacteristic->notify();
      }
    }
  }

  if (newDataReceived) {
    newDataReceived = false;
    M5.Display.fillRect(0, 40, 135, 100, TFT_BLACK);
    M5.Display.setCursor(10, 40);
    M5.Display.setTextColor(TFT_GREEN);
    M5.Display.println("Text from BLE:");
    M5.Display.setCursor(10, 60);
    M5.Display.setTextColor(TFT_WHITE);
    M5.Display.println(receivedText);      // ← String выводится напрямую
  }

  static bool lastConnectionState = false;
  if (deviceConnected != lastConnectionState) {
    lastConnectionState = deviceConnected;
    M5.Display.fillRect(0, 20, 135, 20, TFT_BLACK);
    M5.Display.setCursor(10, 20);
    if (deviceConnected) {
      M5.Display.setTextColor(TFT_GREEN);
      M5.Display.println("Status: Connected");
    } else {
      M5.Display.setTextColor(TFT_RED);
      M5.Display.println("Status: Waiting...");
    }
  }

  delay(10);
}
```