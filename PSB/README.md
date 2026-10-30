#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

const int SENSOR_PIN = A0;

// variabel sinyal
int maxValue;
int minValue;
int peakToPeak;

// kontrol gerakan
int motionCount = 0;
bool motionDetected = false;
bool lastState = false;

// threshold
const int MOTION_THRESHOLD = 20;
const int BASELINE_WINDOW = 50;

// baseline
int baseline = 0;

// MODE
int mode = 0;
// 0 = minta gerak
// 1 = hitung 3 gerakan
// 2 = analisis
// 3 = lock hasil

void setup() {
  Serial.begin(9600);

  lcd.init();
  lcd.backlight();

  lcd.setCursor(0,0);
  lcd.print("VAG SYSTEM");
  lcd.setCursor(0,1);
  lcd.print("STARTING");

  delay(1500);
  lcd.clear();

  // kalibrasi baseline awal
  long sum = 0;
  for(int i=0;i<200;i++){
    sum += analogRead(SENSOR_PIN);
    delay(5);
  }
  baseline = sum / 200;
}

void loop() {

  // =========================
  // MODE 0: instruksi awal
  // =========================
  if (mode == 0) {
    lcd.setCursor(0,0);
    lcd.print("GERAKKAN SENSOR");
    lcd.setCursor(0,1);
    lcd.print("3 KALI");

    motionCount = 0;
    mode = 1;
    delay(1500);
    lcd.clear();
  }

  // =========================
  // MODE 1: hitung gerakan
  // =========================
  if (mode == 1) {

    maxValue = 0;
    minValue = 1023;

    for(int i=0;i<BASELINE_WINDOW;i++){
      int val = analogRead(SENSOR_PIN);

      if(val > maxValue) maxValue = val;
      if(val < minValue) minValue = val;
      delayMicroseconds(200);
    }

    int signal = maxValue - minValue;
    int center = (maxValue + minValue) / 2;

    // deteksi gerakan (edge detection biar tidak double hitung)
    motionDetected = (abs(center - baseline) > MOTION_THRESHOLD);

    if (motionDetected && !lastState) {
      motionCount++;
    }

    lastState = motionDetected;

    lcd.setCursor(0,0);
    lcd.print("GERAK: ");
    lcd.print(motionCount);
    lcd.print("/3   ");

    lcd.setCursor(0,1);
    lcd.print("Lanjutkan...   ");

    if (motionCount >= 3) {
      delay(1000);
      mode = 2;
      lcd.clear();
    }

    delay(200);
  }

  // =========================
  // MODE 2: analisis VAG
  // =========================
  if (mode == 2) {

    maxValue = 0;
    minValue = 1023;

    // ambil data lebih stabil
    for(int i = 0; i < 300; i++) {
      int value = analogRead(SENSOR_PIN);

      if(value > maxValue) maxValue = value;
      if(value < minValue) minValue = value;

      delayMicroseconds(200);
    }

    peakToPeak = maxValue - minValue;

    Serial.print("P2P=");
    Serial.println(peakToPeak);

    lcd.setCursor(0,0);
    lcd.print("HASIL VAG:");
    lcd.print(peakToPeak);
    lcd.print("   ");

    lcd.setCursor(0,1);

    if(peakToPeak < 20) {
      lcd.print("NORMAL        ");
    }
    else if(peakToPeak < 50) {
      lcd.print("OA RINGAN     ");
    }
    else if(peakToPeak < 100) {
      lcd.print("OA SEDANG     ");
    }
    else {
      lcd.print("OA BERAT      ");
    }

    delay(3000);
    mode = 3;
  }

  // =========================
  // MODE 3: LOCK HASIL
  // =========================
  if (mode == 3) {

    lcd.setCursor(0,0);
    lcd.print("HASIL FINAL   ");
    lcd.setCursor(0,1);
    lcd.print("MARI JAGA SENDI ");

    // tidak berubah lagi
    while(true) {
      delay(1000);
    }
  }
}
