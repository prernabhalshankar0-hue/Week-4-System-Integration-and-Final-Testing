
#include <Arduino.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <SPI.h>
#include <MFRC522.h>
#include <BluetoothSerial.h>
#include "HX711.h"

// LCD Configuration
LiquidCrystal_I2C lcd(0x27, 16, 2);

// RFID Configuration
#define SS_PIN 21
#define RST_PIN 22
MFRC522 rfid(SS_PIN, RST_PIN);

// Bluetooth Configuration
BluetoothSerial SerialBT;

// Load Cell Configuration
#define DT 4
#define SCK 5
HX711 scale;

// Buzzer Pin
#define BUZZER 13

// Motor Driver Pins
#define IN1 15
#define IN2 33
#define IN3 27
#define IN4 32

int totalBill = 0;
unsigned long lastCommandTime = 0;
const unsigned long motorTimeout = 2000;

String lastUID = "";
unsigned long lastScanTime = 0;

// Stop Motors
void stopMotors() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
}

// Move Forward
void moveForward() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
}

// Move Backward
void moveBackward() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
}

// Display Bill
void updateDisplay() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Smart Trolley");
  lcd.setCursor(0, 1);
  lcd.print("Bill: Rs.");
  lcd.print(totalBill);
}

// RFID Product Scanning
void scanProduct() {
  if (!rfid.PICC_IsNewCardPresent()) return;
  if (!rfid.PICC_ReadCardSerial()) return;

  String uid = "";

  for (byte i = 0; i < rfid.uid.size; i++) {
    if (rfid.uid.uidByte[i] < 0x10) uid += "0";
    uid += String(rfid.uid.uidByte[i], HEX);
  }

  uid.toUpperCase();

  if (uid != lastUID || millis() - lastScanTime > 1500) {
    totalBill += 50;

    lastUID = uid;
    lastScanTime = millis();

    Serial.println("Product Detected");
    Serial.print("Total Bill: Rs.");
    Serial.println(totalBill);

    tone(BUZZER, 1000, 150);
    updateDisplay();
  }

  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
}

// Bluetooth Control
void bluetoothControl() {
  if (SerialBT.available()) {
    char command = SerialBT.read();

    if (command == 'F') {
      moveForward();
      lastCommandTime = millis();
    }
    else if (command == 'B') {
      moveBackward();
      lastCommandTime = millis();
    }
    else if (command == 'S') {
      stopMotors();
    }
  }
}

// Motor Safety
void motorSafety() {
  if (millis() - lastCommandTime >= motorTimeout) {
    stopMotors();
  }
}

// Load Cell Reading
void readWeight() {
  if (scale.is_ready()) {
    float weight = scale.get_units(1);

    Serial.print("Weight: ");
    Serial.println(weight);
  }
}

void setup() {
  Serial.begin(115200);

  pinMode(BUZZER, OUTPUT);

  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  stopMotors();

  Wire.begin(25, 26);

  lcd.init();
  lcd.backlight();

  SPI.begin();
  rfid.PCD_Init();

  SerialBT.begin("Smart_Trolley");

  scale.begin(DT, SCK);

  lcd.setCursor(0, 0);
  lcd.print("System Starting");
  delay(1000);

  updateDisplay();

  Serial.println("System Integration Started");
}

void loop() {
  scanProduct();
  bluetoothControl();
  motorSafety();
  readWeight();
}
