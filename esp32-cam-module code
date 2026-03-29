#include "esp_camera.h"
#include <WiFi.h>
#include <WebServer.h>

// ===== CAMERA MODEL: AI Thinker ESP32-CAM =====
#define PWDN_GPIO_NUM     32
#define RESET_GPIO_NUM    -1
#define XCLK_GPIO_NUM      0
#define SIOD_GPIO_NUM     26
#define SIOC_GPIO_NUM     27

#define Y9_GPIO_NUM       35
#define Y8_GPIO_NUM       34
#define Y7_GPIO_NUM       39
#define Y6_GPIO_NUM       36
#define Y5_GPIO_NUM       21
#define Y4_GPIO_NUM       19
#define Y3_GPIO_NUM       18
#define Y2_GPIO_NUM        5
#define VSYNC_GPIO_NUM    25
#define HREF_GPIO_NUM     23
#define PCLK_GPIO_NUM     22

#define FLASH_LED_PIN      4

const char* ap_ssid = "ESP32CAM_VIEW";
const char* ap_pass = "12345678";

WebServer server(80);
bool flashOn = false;

void setupCamera() {
  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer   = LEDC_TIMER_0;
  config.pin_d0       = Y2_GPIO_NUM;
  config.pin_d1       = Y3_GPIO_NUM;
  config.pin_d2       = Y4_GPIO_NUM;
  config.pin_d3       = Y5_GPIO_NUM;
  config.pin_d4       = Y6_GPIO_NUM;
  config.pin_d5       = Y7_GPIO_NUM;
  config.pin_d6       = Y8_GPIO_NUM;
  config.pin_d7       = Y9_GPIO_NUM;
  config.pin_xclk     = XCLK_GPIO_NUM;
  config.pin_pclk     = PCLK_GPIO_NUM;
  config.pin_vsync    = VSYNC_GPIO_NUM;
  config.pin_href     = HREF_GPIO_NUM;
  config.pin_sccb_sda = SIOD_GPIO_NUM;
  config.pin_sccb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn     = PWDN_GPIO_NUM;
  config.pin_reset    = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;

  if (psramFound()) {
    config.frame_size   = FRAMESIZE_QQVGA;   // 160x120
    config.jpeg_quality = 12;
    config.fb_count     = 2;
  } else {
    config.frame_size   = FRAMESIZE_QQVGA;
    config.jpeg_quality = 15;
    config.fb_count     = 1;
  }

  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("Camera init failed: 0x%x\n", err);
    while (true) delay(1000);
  }

  sensor_t* s = esp_camera_sensor_get();
  s->set_brightness(s, 0);
  s->set_contrast(s, 0);
  s->set_saturation(s, 0);
}

void handleRoot() {
  String html =
    "<html><body style='font-family:Arial;background:#111;color:#fff;text-align:center;'>"
    "<h2>ESP32-CAM Viewer</h2>"
    "<p>AP: ESP32CAM_VIEW</p>"
    "<p><a href='/jpg' style='color:#0f0;'>/jpg</a></p>"
    "<p><a href='/flash?led=1' style='color:#ff0;'>Flash ON</a> | "
    "<a href='/flash?led=0' style='color:#ff0;'>Flash OFF</a></p>"
    "</body></html>";
  server.send(200, "text/html", html);
}

void handleJpg() {
  camera_fb_t* fb = esp_camera_fb_get();
  if (!fb) {
    server.send(500, "text/plain", "Camera capture failed");
    return;
  }

  server.setContentLength(fb->len);
  server.send(200, "image/jpeg", "");
  WiFiClient client = server.client();
  client.write(fb->buf, fb->len);

  esp_camera_fb_return(fb);
}

void handleFlash() {
  if (!server.hasArg("led")) {
    server.send(400, "text/plain", "Missing led arg");
    return;
  }

  flashOn = (server.arg("led") == "1");
  digitalWrite(FLASH_LED_PIN, flashOn ? HIGH : LOW);
  server.send(200, "text/plain", flashOn ? "FLASH ON" : "FLASH OFF");
}

void setup() {
  Serial.begin(115200);
  pinMode(FLASH_LED_PIN, OUTPUT);
  digitalWrite(FLASH_LED_PIN, LOW);

  setupCamera();

  WiFi.mode(WIFI_AP);
  bool ok = WiFi.softAP(ap_ssid, ap_pass);
  if (!ok) {
    Serial.println("AP start failed");
    while (true) delay(1000);
  }

  Serial.println("ESP32-CAM AP started");
  Serial.print("AP IP: ");
  Serial.println(WiFi.softAPIP());

  server.on("/", handleRoot);
  server.on("/jpg", HTTP_GET, handleJpg);
  server.on("/flash", HTTP_GET, handleFlash);
  server.begin();
}

void loop() {
  server.handleClient();
}
