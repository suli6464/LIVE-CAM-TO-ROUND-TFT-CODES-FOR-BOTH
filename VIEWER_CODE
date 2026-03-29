#include <WiFi.h>
#include <HTTPClient.h>
#include <SPI.h>
#include <Adafruit_GFX.h>
#include <Adafruit_GC9A01A.h>
#include <TJpg_Decoder.h>

#define TFT_CS   5
#define TFT_DC   2
#define TFT_RST  4

#define BTN_CONNECT 27
#define BTN_FLASH   26

Adafruit_GC9A01A tft(TFT_CS, TFT_DC, TFT_RST);

const char* cam_ssid = "ESP32CAM_VIEW";
const char* cam_pass = "12345678";
const char* cam_jpg_url   = "http://192.168.4.1/jpg";
const char* cam_flash_on  = "http://192.168.4.1/flash?led=1";
const char* cam_flash_off = "http://192.168.4.1/flash?led=0";

bool camConnected = false;
bool flashState = false;

unsigned long lastFrameMs = 0;
unsigned long lastConnectTry = 0;
unsigned long lastBtnMs = 0;

uint8_t* jpgBuffer = nullptr;
size_t jpgBufferSize = 0;

// image position for 160x120 centered on 240x240
const int IMG_W = 160;
const int IMG_H = 120;
const int IMG_X = (240 - IMG_W) / 2;
const int IMG_Y = (240 - IMG_H) / 2;

bool tftOutput(int16_t x, int16_t y, uint16_t w, uint16_t h, uint16_t* bitmap) {
  tft.drawRGBBitmap(IMG_X + x, IMG_Y + y, bitmap, w, h);
  return true;
}

bool buttonPressed(uint8_t pin) {
  return digitalRead(pin) == LOW;
}

void drawStatus(const String& line1, const String& line2 = "") {
  tft.fillScreen(tft.color565(0, 0, 0));
  tft.setTextColor(tft.color565(255, 255, 255));
  tft.setTextSize(2);
  tft.setCursor(30, 70);
  tft.print("CAM VIEWER");
  tft.setTextSize(1);
  tft.setCursor(20, 130);
  tft.print(line1);
  if (line2.length()) {
    tft.setCursor(20, 145);
    tft.print(line2);
  }
  tft.setCursor(20, 190);
  tft.print("BTN27: connect");
  tft.setCursor(20, 205);
  tft.print("BTN26: flash");
}

void drawOverlay() {
  tft.fillRect(0, 0, 240, 16, tft.color565(0, 0, 0));
  tft.setTextSize(1);
  tft.setTextColor(tft.color565(255, 255, 255));
  tft.setCursor(6, 4);
  tft.print(camConnected ? "CAM LINKED" : "NO LINK");

  tft.setCursor(160, 4);
  tft.print(flashState ? "FLASH ON" : "FLASH OFF");
}

bool connectToCamAP() {
  WiFi.mode(WIFI_STA);
  WiFi.disconnect(true, true);
  delay(200);

  WiFi.begin(cam_ssid, cam_pass);

  unsigned long start = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - start < 8000) {
    delay(250);
  }

  camConnected = (WiFi.status() == WL_CONNECTED);
  return camConnected;
}

bool httpGetSmall(const char* url) {
  HTTPClient http;
  http.setTimeout(3000);
  if (!http.begin(url)) return false;

  int code = http.GET();
  bool ok = (code == 200);
  http.end();
  return ok;
}

bool fetchAndShowFrame() {
  if (!camConnected) return false;

  HTTPClient http;
  http.setTimeout(4000);

  if (!http.begin(cam_jpg_url)) {
    return false;
  }

  int code = http.GET();
  if (code != 200) {
    http.end();
    return false;
  }

  int len = http.getSize();
  if (len <= 0 || len > 40000) {
    http.end();
    return false;
  }

  if (jpgBufferSize < (size_t)len) {
    if (jpgBuffer) free(jpgBuffer);
    jpgBuffer = (uint8_t*)ps_malloc(len);
    if (!jpgBuffer) jpgBuffer = (uint8_t*)malloc(len);
    if (!jpgBuffer) {
      jpgBufferSize = 0;
      http.end();
      return false;
    }
    jpgBufferSize = len;
  }

  WiFiClient* stream = http.getStreamPtr();
  int index = 0;

  while (http.connected() && index < len) {
    int availableBytes = stream->available();
    if (availableBytes > 0) {
      int toRead = min(availableBytes, len - index);
      int readNow = stream->readBytes((char*)(jpgBuffer + index), toRead);
      if (readNow > 0) index += readNow;
    } else {
      delay(1);
    }
  }

  http.end();

  if (index != len) return false;

  tft.fillRect(IMG_X, IMG_Y, IMG_W, IMG_H, tft.color565(0, 0, 0));
  TJpgDec.drawJpg(0, 0, jpgBuffer, len);
  drawOverlay();
  return true;
}

void setup() {
  Serial.begin(115200);

  pinMode(BTN_CONNECT, INPUT_PULLUP);
  pinMode(BTN_FLASH, INPUT_PULLUP);

  tft.begin();
  tft.setRotation(0);

  TJpgDec.setJpgScale(1);
  TJpgDec.setCallback(tftOutput);

  drawStatus("Press connect button", "to join ESP32-CAM WiFi");
}

void loop() {
  if (millis() - lastBtnMs > 220) {
    if (buttonPressed(BTN_CONNECT)) {
      lastBtnMs = millis();
      drawStatus("Connecting to camera AP...");
      if (connectToCamAP()) {
        drawStatus("Connected!", WiFi.localIP().toString());
        delay(600);
        tft.fillScreen(tft.color565(0, 0, 0));
        drawOverlay();
      } else {
        drawStatus("Connect failed", "Press connect again");
      }
    }

    if (buttonPressed(BTN_FLASH)) {
      lastBtnMs = millis();
      if (camConnected) {
        flashState = !flashState;
        bool ok = httpGetSmall(flashState ? cam_flash_on : cam_flash_off);
        if (!ok) {
          flashState = !flashState;
        }
        drawOverlay();
      }
    }
  }

  // keep pulling frames
  if (camConnected && millis() - lastFrameMs > 250) { // ~4 FPS target
    lastFrameMs = millis();

    bool ok = fetchAndShowFrame();
    if (!ok) {
      camConnected = false;
      drawStatus("Stream lost", "Press connect button");
    }
  }
}
