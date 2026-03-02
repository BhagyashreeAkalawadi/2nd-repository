// Health Monitoring System using NodeMCU + MAX30102 + 16x2 LCD
// Displays Heart Rate (BPM) on LCD

#include <Wire.h>
#include "MAX30105.h"
#include "heartRate.h"
#include <LiquidCrystal_I2C.h>

MAX30105 particleSensor;
LiquidCrystal_I2C lcd(0x27, 16, 2);  // Use 0x3F if 0x27 doesn't work

const byte RATE_SIZE = 4; // Average heart rate over 4 samples
byte rates[RATE_SIZE];    // Array to hold BPM readings
byte rateSpot = 0;
long lastBeat = 0;        // Time when last beat occurred
float beatsPerMinute;
int beatAvg;

void setup() {
  Serial.begin(115200);
  Wire.begin(D2, D1);  // SDA, SCL pins for NodeMCU

  lcd.init();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Health Monitor");
  lcd.setCursor(0, 1);
  lcd.print("Initializing...");
  delay(2000);
  lcd.clear();

  if (!particleSensor.begin(Wire, I2C_SPEED_STANDARD)) {
    lcd.print("MAX30102 Error!");
    Serial.println("MAX30102 not found. Check wiring!");
    while (1);
  }

  particleSensor.setup(); // Use default setup
  particleSensor.setPulseAmplitudeRed(0x0A); // Adjust LED brightness
  particleSensor.setPulseAmplitudeIR(0x0A);
  particleSensor.setSampleRate(100);
  particleSensor.enableDIETEMPRDY(); // Enable internal temperature sensor

  lcd.clear();
  lcd.print("Place finger on");
  lcd.setCursor(0, 1);
  lcd.print("sensor...");
  delay(3000);
  lcd.clear();
}

void loop() {
  long irValue = particleSensor.getIR(); // Get IR value

  if (checkForBeat(irValue)) {  // If a heartbeat is detected
    long delta = millis() - lastBeat;
    lastBeat = millis();

    beatsPerMinute = 60 / (delta / 1000.0);

    if (beatsPerMinute < 255 && beatsPerMinute > 30) {
      rates[rateSpot++] = (byte)beatsPerMinute;
      rateSpot %= RATE_SIZE;

      long sum = 0;
      for (byte x = 0; x < RATE_SIZE; x++)
        sum += rates[x];
      beatAvg = sum / RATE_SIZE;
    }
  }

  lcd.setCursor(0, 0);
  lcd.print("BPM: ");
  if (beatAvg > 0) {
    lcd.print(beatAvg);
    lcd.print("   ");
  } else {
    lcd.print("--  ");
  }

  lcd.setCursor(0, 1);
  lcd.print("IR: ");
  lcd.print(irValue);
  lcd.print("   ");

  Serial.print("IR=");
  Serial.print(irValue);
  Serial.print("\tBPM=");
  Serial.println(beatAvg);

  delay(200);
}
