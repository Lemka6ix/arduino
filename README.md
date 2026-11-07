# arduino

## 1
```C
#include <M5Unified.h>
#include <WiFi.h>
#include "time.h"
#include <math.h>

const char* ssid     = "Имя_вашей_сети";
const char* password = "Пароль_вашей_сети";

const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = 18000;
const int   daylightOffset_sec = 0;

int cx, cy;
const int R_FACE = 55;
const int R_HOUR = 28;
const int R_MIN  = 40;
const int R_SEC  = 48;

int lastSec = -1;
int lastMin = -1;
int lastHour = -1;
int lastDay = -1;

void drawFace() {
  int x0 = cx - R_FACE - 3;
  int y0 = cy - R_FACE - 3;
  int w  = 2 * (R_FACE + 3);
  int h  = 2 * (R_FACE + 3);
  M5.Display.fillRect(x0, y0, w, h, TFT_BLACK);

  M5.Display.drawCircle(cx, cy, R_FACE, TFT_WHITE);
  M5.Display.drawCircle(cx, cy, R_FACE - 1, TFT_WHITE);

  for (int i = 0; i < 12; i++) {
    float a = i * 30.0f * PI / 180.0f;
    int inner = (i % 3 == 0) ? 12 : 7;
    int x1 = cx + (R_FACE - 3) * sinf(a);
    int y1 = cy - (R_FACE - 3) * cosf(a);
    int x2 = cx + (R_FACE - inner) * sinf(a);
    int y2 = cy - (R_FACE - inner) * cosf(a);
    M5.Display.drawLine(x1, y1, x2, y2, TFT_WHITE);
  }

  M5.Display.fillCircle(cx, cy, 3, TFT_WHITE);
}

void drawHands(int h, int m, int s) {
  float ah = ((h % 12) + m / 60.0f) * 30.0f * PI / 180.0f;
  float am = (m + s / 60.0f) * 6.0f * PI / 180.0f;
  float as = s * 6.0f * PI / 180.0f;

  int xh = cx + R_HOUR * sinf(ah);
  int yh = cy - R_HOUR * cosf(ah);
  int xm = cx + R_MIN  * sinf(am);
  int ym = cy - R_MIN  * cosf(am);
  int xs = cx + R_SEC  * sinf(as);
  int ys = cy - R_SEC  * cosf(as);

  M5.Display.drawLine(cx, cy, xh, yh, TFT_RED);
  M5.Display.drawLine(cx, cy, xm, ym, TFT_WHITE);
  M5.Display.drawLine(cx, cy, xs, ys, TFT_GREEN);

  M5.Display.fillCircle(cx, cy, 3, TFT_WHITE);
}

void drawDate(const struct tm& t) {
  M5.Display.setTextDatum(middle_center);
  M5.Display.setTextColor(TFT_GREEN, TFT_BLACK);
  M5.Display.setFont(&fonts::FreeMonoBold9pt7b);

  char buf[16];
  snprintf(buf, sizeof(buf), "%02d/%02d/%04d",
           t.tm_mday, t.tm_mon + 1, t.tm_year + 1900);
  M5.Display.drawString(buf, cx, 15);
}

void drawTimeText(const struct tm& t) {
  M5.Display.setTextDatum(middle_center);
  M5.Display.setTextColor(TFT_GREEN, TFT_BLACK);
  M5.Display.setFont(&fonts::FreeMonoBold12pt7b);

  char buf[16];
  snprintf(buf, sizeof(buf), "%02d:%02d:%02d",
           t.tm_hour, t.tm_min, t.tm_sec);
  M5.Display.drawString(buf, cx, 215);
}

void setup() {
  M5.begin();

  M5.Display.setRotation(0);
  M5.Display.fillScreen(TFT_BLACK);

  cx = M5.Display.width() / 2;
  cy = 115;

  M5.Display.setTextDatum(middle_center);
  M5.Display.setTextColor(TFT_WHITE, TFT_BLACK);
  M5.Display.setFont(&fonts::FreeMonoBold9pt7b);
  M5.Display.drawString("WiFi...", cx, cy);

  Serial.begin(115200);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println();
  Serial.println("WiFi connected");
  Serial.println(WiFi.localIP());

  M5.Display.fillScreen(TFT_BLACK);
  M5.Display.drawString("NTP sync...", cx, cy);

  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);

  struct tm t;
  unsigned long start = millis();
  while (!getLocalTime(&t) && (millis() - start < 15000)) {
    delay(200);
  }

  drawFace();
  if (getLocalTime(&t)) {
    drawDate(t);
    lastDay = t.tm_mday;
  }
}

void loop() {
  M5.update();

  struct tm t;
  if (!getLocalTime(&t)) {
    M5.Display.setTextDatum(middle_center);
    M5.Display.setTextColor(TFT_RED, TFT_BLACK);
    M5.Display.setFont(&fonts::FreeMonoBold9pt7b);
    M5.Display.drawString("No NTP", cx, 215);
    delay(500);
    return;
  }

  if (t.tm_mday != lastDay) {
    lastDay = t.tm_mday;
    drawDate(t);
  }

  if (t.tm_sec != lastSec) {
    lastSec  = t.tm_sec;
    lastMin  = t.tm_min;
    lastHour = t.tm_hour;

    drawFace();
    drawHands(t.tm_hour, t.tm_min, t.tm_sec);
    drawTimeText(t);
  }

  delay(50);
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
