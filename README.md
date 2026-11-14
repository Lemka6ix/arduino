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

const char* ssid = "Имя_вашей_сети";
const char* password = "Пароль_вашей_сети";

WebServer server(80);

bool rect1On = false;
bool rect2On = false;

void drawScreen() {
  M5.Display.setRotation(0);
  M5.Display.clear(TFT_BLACK);
  M5.Display.setTextSize(1);
  M5.Display.setTextColor(TFT_WHITE, TFT_BLACK);

  M5.Display.drawString("WEB CONTROL", 10, 5);

  M5.Display.drawString(String("Rect1: ") + (rect1On ? "ON" : "OFF"), 10, 30);
  if (rect1On) M5.Display.fillRect(20, 50, 95, 35, TFT_RED);
  else         M5.Display.fillRect(20, 50, 95, 35, TFT_DARKGREY);

  M5.Display.drawString(String("Rect2: ") + (rect2On ? "ON" : "OFF"), 10, 100);
  if (rect2On) M5.Display.fillRect(20, 120, 95, 35, TFT_GREEN);
  else         M5.Display.fillRect(20, 120, 95, 35, TFT_DARKGREY);

  M5.Display.setTextColor(TFT_YELLOW, TFT_BLACK);
  M5.Display.drawString("Open in browser:", 10, 190);
  M5.Display.drawString(WiFi.localIP().toString(), 10, 210);
}

void handleRoot() {
  String page;
  page.reserve(1500);

  page += "<!DOCTYPE html><html><head>";
  page += "<meta charset='utf-8'>";
  page += "<meta name='viewport' content='width=device-width, initial-scale=1'>";
  page += "<title>M5StickS3 Control</title>";
  page += "<style>";
  page += "body{font-family:Arial;margin:24px;}";
  page += "button{font-size:18px;padding:10px 18px;margin:6px 0;";
  page += "cursor:pointer;color:#fff;border:none;border-radius:6px;}";
  page += ".btn1{background:#d32f2f;}";
  page += ".btn2{background:#2e7d32;}";
  page += ".state{font-size:18px;margin-top:16px;}";
  page += "</style></head><body>";

  page += "<h2>Управление прямоугольниками</h2>";

  page += "<p class='state'>Прямоугольник 1: <b>";
  page += rect1On ? "ВКЛ" : "ВЫКЛ";
  page += "</b></p>";
  page += "<form action='/toggle1' method='get'>";
  page += "<button class='btn1' type='submit'>Переключить прямоугольник 1</button>";
  page += "</form>";

  page += "<p class='state'>Прямоугольник 2: <b>";
  page += rect2On ? "ВКЛ" : "ВЫКЛ";
  page += "</b></p>";
  page += "<form action='/toggle2' method='get'>";
  page += "<button class='btn2' type='submit'>Переключить прямоугольник 2</button>";
  page += "</form>";

  page += "</body></html>";

  server.send(200, "text/html; charset=utf-8", page);
}

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

  M5.Display.setRotation(0);
  M5.Display.fillScreen(TFT_BLACK);
  M5.Display.setTextColor(TFT_WHITE, TFT_BLACK);
  M5.Display.drawString("Connecting to WiFi...", 10, 10);

  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println();
  Serial.print("IP: ");
  Serial.println(WiFi.localIP());

  drawScreen();

  server.on("/", handleRoot);
  server.on("/toggle1", handleToggle1);
  server.on("/toggle2", handleToggle2);
  server.onNotFound(handleNotFound);

  server.begin();
}

void loop() {
  M5.update();
  server.handleClient();

  static unsigned long lastIpPrint = 0;
  if (millis() - lastIpPrint > 5000) {
    lastIpPrint = millis();
    Serial.print("IP: ");
    Serial.println(WiFi.localIP());
  }

  delay(2);
}
```

 ```C
void handleRoot() {
  String page;
  page.reserve(1700);

  page += "<!DOCTYPE html><html><head>";
  page += "<meta charset='utf-8'>";
  page += "<meta name='viewport' content='width=device-width, initial-scale=1'>";
  page += "<title>M5StickS3 Control</title>";
  page += "<style>";
  page += "body{font-family:Arial;margin:24px;}";
  page += "button{font-size:18px;padding:10px 18px;margin:6px 0;";
  page += "cursor:pointer;color:#fff;border:none;border-radius:6px;}";
  page += ".btn1{background:#d32f2f;}";        /* красная */
  page += ".btn2{background:#2e7d32;}";        /* зелёная */
  page += ".btn1_gray{background:#9e9e9e;}";   /* серая (вкл) */
  page += ".btn2_gray{background:#9e9e9e;}";   /* серая (вкл) */
  page += ".state{font-size:18px;margin-top:16px;}";
  page += "</style></head><body>";

  page += "<h2>Управление прямоугольниками</h2>";

  /* Прямоугольник 1 */
  page += "<p class='state'>Прямоугольник 1: <b>";
  page += rect1On ? "ВКЛ" : "ВЫКЛ";
  page += "</b></p>";
  page += "<form action='/toggle1' method='get'>";
  page += "<button class='";
  page += rect1On ? "btn1_gray" : "btn1";
  page += "' type='submit'>Переключить прямоугольник 1</button>";
  page += "</form>";

  /* Прямоугольник 2 */
  page += "<p class='state'>Прямоугольник 2: <b>";
  page += rect2On ? "ВКЛ" : "ВЫКЛ";
  page += "</b></p>";
  page += "<form action='/toggle2' method='get'>";
  page += "<button class='";
  page += rect2On ? "btn2_gray" : "btn2";
  page += "' type='submit'>Переключить прямоугольник 2</button>";
  page += "</form>";

  page += "</body></html>";

  server.send(200, "text/html; charset=utf-8", page);
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

  float dx = ax * SCALE * AX_SIGN;
  float dy = ay * SCALE * AY_SIGN;

  float maxDx = CX - BALL_R - 2;
  float maxDy = CY - BALL_R - 2;
  dx = constrain(dx, -maxDx, maxDx);
  dy = constrain(dy, -maxDy, maxDy);

  int bx = CX + (int)dx;
  int by = CY + (int)dy;

  M5.Display.fillCircle(bx, by, BALL_R, TFT_YELLOW);

  float tilt = sqrt(ax * ax + ay * ay);
  bool level = (tilt < LEVEL_THRESHOLD);

  M5.Display.setTextSize(1);
  M5.Display.setTextColor(level ? TFT_GREEN : TFT_WHITE, TFT_BLACK);
  M5.Display.drawString(level ? "LEVEL OK" : "TILT", 8, 8);
}

void setup() {
  M5.begin();

  M5.Display.setRotation(0);

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


## 11
```C
#include <M5Unified.h>
#include <math.h>

bool alarmActive = false;

float base_ax = 0, base_ay = 0, base_az = 0;
float base_norm = 1.0;

const float ANGLE_THRESHOLD = 10.0;  // градусы
const float GYRO_THRESHOLD  = 25.0;  // град/с

// Переменные для звука
unsigned long previousToneMillis = 0;
int toneState = 0;
const int TONE_DURATION = 150; // мс
const int PAUSE_DURATION = 50; // мс

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
      M5.Speaker.stop();          // останавливаем звук
      toneState = 0;              // сбрасываем фазу тона
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
        M5.Speaker.tone(4000, TONE_DURATION);
        toneState = 1;
      } else {
        M5.Speaker.tone(7000, TONE_DURATION);
        toneState = 0;
      }
    }
  } else {
    delay(10);
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
