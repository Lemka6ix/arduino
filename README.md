# arduino

## 1
```C
#include "M5Unified.h"
#include <WiFi.h>

const char* ssid = "ASOIU";
const char* password = "kaf.asoiu.48";

void setup() {
  // put your setup code here, to run once:
  M5.begin();
  M5.Display.setRotation(0);
  M5.Display.fillScreen(TFT_BLACK);
  M5.Display.setTextColor(TFT_WHITE);
  M5.Display.setTextSize(1);

  Serial.begin(115200);
  delay(1000);

  M5.Display.setCursor(0, 0);
  M5.Display.println("WiFi Connecting..")

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  Serial.print("подключение к WiFi");
  M5.Display.print("Connecting");

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
    M5.Display.print(".");
  }

  Serial.println("");
  Serial.println("WiFi подключен!");
  Serial.print("IP-адресс: ");
  Serial.println(WiFi.localIP());

  M5.Display.fillScreen(TFT_BLACK);
  M5.Display.setCursor(0, 0);

  M5.Display.setTextColor(TFT_GREEN);
  M5.Display.println("WiFi connected");

  M5.Display.setTextColor(TFT_WHITE);
  M5.Display.print("IP: ");
  M5.Display.println(WiFi.localIP());
}

void loop() {
  // put your main code here, to run repeatedly:
  M5.update();
  delay(100);
}

```


## 2
```C
#include <M5Unified.h>
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "ASOIU";
const char* password = "kaf.asoiu.48";

WebServer server(80);

bool rect1On = false; // первый прямоугольник — красный
bool rect2On = false; // второй прямоугольник — зелёный

// Отрисовка экрана M5StickS3
void drawScreen() {
  M5.Display.setRotation(1);   // горизонтальная ориентация, 240x135
  M5.Display.clear(TFT_BLACK);
  M5.Display.setTextSize(1);
  M5.Display.setTextColor(TFT_WHITE, TFT_BLACK);

  M5.Display.drawString("WEB CONTROL", 10, 8);

  // Текст состояния над прямоугольниками
  M5.Display.drawString(String("Rect1: ") + (rect1On ? "ON" : "OFF"), 10, 32);
  M5.Display.drawString(String("Rect2: ") + (rect2On ? "ON" : "OFF"), 130, 32);

  // Первый прямоугольник
  if (rect1On) {
    M5.Display.fillRect(20, 60, 80, 45, TFT_RED);
  } else {
    M5.Display.fillRect(20, 60, 80, 45, TFT_DARKGREY);
  }

  // Второй прямоугольник
  if (rect2On) {
    M5.Display.fillRect(140, 60, 80, 45, TFT_GREEN);
  } else {
    M5.Display.fillRect(140, 60, 80, 45, TFT_DARKGREY);
  }
}

// Главная страница Web-сервера
void handleRoot() {
  String page;
  page.reserve(1200);

  page += "<!DOCTYPE html><html><head>";
  page += "<meta charset='utf-8'>";
  page += "<meta name='viewport' content='width=device-width, initial-scale=1'>";
  page += "<title>M5StickS3 Control</title>";
  page += "<style>";
  page += "body{font-family:Arial;margin:24px;}";
  page += "button{font-size:18px;padding:10px 18px;margin:6px 0;cursor:pointer;}";
  page += ".state{font-size:18px;margin-top:16px;}";
  page += "</style></head><body>";

  page += "<h2>Управление прямоугольниками</h2>";

  page += "<p class='state'>Прямоугольник 1: <b>";
  page += rect1On ? "ВКЛ" : "ВЫКЛ";
  page += "</b></p>";
  page += "<form action='/toggle1' method='get'>";
  page += "<button type='submit'>Переключить прямоугольник 1</button>";
  page += "</form>";

  page += "<p class='state'>Прямоугольник 2: <b>";
  page += rect2On ? "ВКЛ" : "ВЫКЛ";
  page += "</b></p>";
  page += "<form action='/toggle2' method='get'>";
  page += "<button type='submit'>Переключить прямоугольник 2</button>";
  page += "</form>";

  page += "</body></html>";

  server.send(200, "text/html; charset=utf-8", page);
}

// Обработчики переключения
void handleToggle1() {
  rect1On = !rect1On;
  drawScreen();

  server.sendHeader("Location", "/");
  server.send(303, "text/plain", "");
}

void handleToggle2() {
  rect2On = !rect2On;
  drawScreen();

  server.sendHeader("Location", "/");
  server.send(303, "text/plain", "");
}

void handleNotFound() {
  server.send(404, "text/plain", "Not found");
}

void setup() {
  Serial.begin(115200);

  M5.begin();
  M5.Display.setRotation(1);
  M5.Display.clear(TFT_BLACK);
  M5.Display.setTextColor(TFT_WHITE, TFT_BLACK);
  M5.Display.drawString("Connecting to WiFi...", 10, 10);

  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
    M5.Display.drawString(".", 10 + (millis() / 500) % 200, 30);
  }

  Serial.println();
  Serial.println("WiFi connected");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());

  drawScreen();

  
  M5.Display.fillRect(0, 110, 240, 25, TFT_BLACK);
  M5.Display.drawString(WiFi.localIP().toString(), 10, 112);

  server.on("/", handleRoot);
  server.on("/toggle1", handleToggle1);
  server.on("/toggle2", handleToggle2);
  server.onNotFound(handleNotFound);

  server.begin();
  Serial.println("Web server started");
}

void loop() {
  M5.update();
  server.handleClient();
  delay(2);
}
```

## 3
```C
#include <M5Unified.h>

int CX;
int CY;                 
const int TARGET_R = 32; 
const int BALL_R   = 8;  

const float SCALE = 90.0;

const float AX_SIGN = 1.0;
const float AY_SIGN = 1.0;

const float LEVEL_THRESHOLD = 0.08;

void drawLevel(float ax, float ay) {
  M5.Display.fillScreen(TFT_BLACK);

  M5.Display.drawCircle(CX, CY, TARGET_R, TFT_GREEN);

  M5.Display.drawLine(CX - TARGET_R, CY, CX + TARGET_R, CY, TFT_DARKGREY);
  M5.Display.drawLine(CX, CY - TARGET_R, CX, CY + TARGET_R, TFT_DARKGREY);

  // Смещение жёлтого кружка от наклона
  float dx = ax * SCALE * AX_SIGN;
  float dy = ay * SCALE * AY_SIGN;

  // Ограничиваем, чтобы кружок не ушёл за экран
  float maxDx = CX - BALL_R - 2;
  float maxDy = CY - BALL_R - 2;
  dx = constrain(dx, -maxDx, maxDx);
  dy = constrain(dy, -maxDy, maxDy);

  int bx = CX + (int)dx;
  int by = CY + (int)dy;

  M5.Display.fillCircle(bx, by, BALL_R, TFT_YELLOW);

  // Индикация горизонтального положения
  float tilt = sqrt(ax * ax + ay * ay);
  bool level = (tilt < LEVEL_THRESHOLD);

  M5.Display.setTextSize(1);
  M5.Display.setTextColor(level ? TFT_GREEN : TFT_WHITE, TFT_BLACK);
  M5.Display.drawString(level ? "LEVEL OK" : "TILT", 8, 8);
}

void setup() {
  M5.begin();

  // Горизонтальная ориентация
  M5.Display.setRotation(1);

  CX = M5.Display.width() / 2;
  CY = M5.Display.height() / 2;

  M5.Display.fillScreen(TFT_BLACK);
  M5.Display.setTextColor(TFT_WHITE, TFT_BLACK);
  M5.Display.drawString("IMU LEVEL", 8, 8);
}

void loop() {
  M5.update();

  if (M5.Imu.update()) {
    auto imu = M5.Imu.getImuData();

    drawLevel(imu.accel.x, imu.accel.y);
  }

  delay(20);
}
```
