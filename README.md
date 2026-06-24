#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <SoftwareSerial.h>
#include <TinyGPS++.h>

#define SCREEN_WIDTH 128 
#define SCREEN_HEIGHT 32 

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

static const int RXPin = 4, TXPin = 3;
static const int BUTTON_PIN = 2; // Touch switch connected to Pin 2
static const uint32_t GPSBaud = 9600;

TinyGPSPlus gps;
SoftwareSerial ss(RXPin, TXPin);

int screenMode = 0; // 0: Lat, 1: Lng, 2: Status, 3: Time, 4: Date
bool lastButtonState = HIGH;

void setup() {
  Serial.begin(115200); 
  ss.begin(GPSBaud);
  
  pinMode(BUTTON_PIN, INPUT_PULLUP);

  if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) { 
    Serial.println(F("SSD1306 allocation failed"));
    for(;;);
  }
  
  display.clearDisplay();
  display.setTextSize(2); 
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(15, 8);
  display.println("STARTING");
  display.display();
  delay(2000);
}

void loop() {
  // Background mein GPS data lagatar update hota rahega
  while (ss.available() > 0) {
    gps.encode(ss.read());
  }

  // --- ONLY SINGLE CLICK LOGIC ---
  bool currentButtonState = digitalRead(BUTTON_PIN);

  // Jab touch detect ho aur pichli baar state HIGH thi (Click Event)
  if (lastButtonState == HIGH && currentButtonState == LOW) {
    screenMode = (screenMode + 1) % 5; // Screen change karein (0 se 4 tak)
    
    delay(300); // Debounce Delay: Taaki touch chhodte waqt double click register na ho
  }
  
  lastButtonState = currentButtonState;

  // Display Render Engine
  if (gps.location.isValid()) {
    displayGPSData();
  } else {
    display.clearDisplay();
    display.setTextSize(2);
    display.setCursor(20, 8);
    display.println("NO LOCK");
    display.display();
    
    Serial.print("Waiting for Lock... Sats: ");
    Serial.println(gps.satellites.value());
  }
}

void displayGPSData() {
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);

  if (screenMode == 0) {
    // ---- 1. LATITUDE ----
    display.setTextSize(2); 
    display.setCursor(0, 8);
    display.print("LT:"); 
    display.print(gps.location.lat(), 4);
    Serial.print("Latitude: "); Serial.println(gps.location.lat(), 6);

  } else if (screenMode == 1) {
    // ---- 2. LONGITUDE ----
    display.setTextSize(2); 
    display.setCursor(0, 8);
    display.print("LN:"); 
    display.print(gps.location.lng(), 4);
    Serial.print("Longitude: "); Serial.println(gps.location.lng(), 6);

  } else if (screenMode == 2) {
    // ---- 3. SPEED & SATELLITES ----
    display.setTextSize(2); 
    display.setCursor(0, 0);
    display.print(gps.speed.kmph(), 0); display.println(" km/h");
    display.print("SATS: "); display.println(gps.satellites.value());
    Serial.print("Speed: "); Serial.print(gps.speed.kmph()); Serial.print(" km/h | Sats: "); Serial.println(gps.satellites.value());

  } else if (screenMode == 3) {
    // ---- 4. TIME WITH LIVE SECONDS ----
    int hour = gps.time.hour() + 5;
    int minute = gps.time.minute() + 30;
    int second = gps.time.second();

    if (minute >= 60) { minute -= 60; hour += 1; }
    if (hour >= 24) { hour -= 24; }

    String ampm = "AM";
    if (hour >= 12) { ampm = "PM"; if (hour > 12) hour -= 12; }
    if (hour == 0) hour = 12;

    display.setTextSize(2); 
    display.setCursor(0, 0);
    if (hour < 10) display.print("0");
    display.print(hour); display.print(":");
    if (minute < 10) display.print("0");
    display.print(minute); display.print(":");
    if (second < 10) display.print("0");
    display.print(second);
    
    display.setCursor(45, 18);
    display.setTextSize(1.5); 
    display.print(ampm);

    Serial.print("Time: "); Serial.print(hour); Serial.print(":"); Serial.print(minute); Serial.print(":"); Serial.print(second); Serial.println(" " + ampm);

  } else if (screenMode == 4) {
    // ---- 5. DATE ----
    int day = gps.date.day();
    int month = gps.date.month();
    int year = gps.date.year();

    display.setTextSize(2); 
    display.setCursor(0, 8); 
    if (day < 10) display.print("0");
    display.print(day); display.print("/");
    if (month < 10) display.print("0");
    display.print(month); display.print("/");
    display.print(year - 2000); 

    Serial.print("Date: "); Serial.print(day); Serial.print("/"); Serial.print(month); Serial.print("/"); Serial.println(year);
  }

  display.display();
}
