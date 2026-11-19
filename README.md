# arduino

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
