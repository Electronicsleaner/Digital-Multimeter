/*
  Digital Multimeter with Arduino and OLED
  Functions:
  1. Resistance Measurement
  2. Voltage Measurement
  3. Current Measurement
  4. Capacitance Measurement

  OLED: SSD1306 128x32 I2C
  Arduino: Arduino Uno
*/

#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 32
#define OLED_RESET -1

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// -------------------- Pins --------------------

const int SELECT_BUTTON = 2;
const int RIGHT_BUTTON  = 3;

const int R1_PIN = 4;
const int R2_PIN = 5;
const int R3_PIN = 6;

const int OHM_METER         = A0;
const int CAPACITANCE_METER = A1;
const int VOLT_METER        = A2;
const int AMMETER           = A3;

const int CHARGE_PIN    = 13;
const int DISCHARGE_PIN = 11;

// -------------------- Variables --------------------

bool isSelected = false;

int navigator = 0;

float resistance = 0.0;
float voltage = 0.0;
float current = 0.0;
float capacitance = 0.0;

bool kiloOhm = false;
bool milliAmp = false;
bool nanoFarad = false;

// ==================================================
// OLED FUNCTIONS
// ==================================================

void OLED_init()
{
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C))
  {
    Serial.println("OLED initialization failed!");

    while (1);
  }

  display.clearDisplay();

  display.setTextColor(WHITE);

  display.setTextSize(2);
  display.setCursor(20, 2);
  display.println("DIGITAL");

  display.setCursor(30, 18);
  display.println("METER");

  display.display();

  delay(2000);

  display.clearDisplay();
  display.display();
}


void displayClear()
{
  display.clearDisplay();
}


void displayText(int size, int x, int y, String text)
{
  display.setTextSize(size);
  display.setTextColor(WHITE);
  display.setCursor(x, y);
  display.println(text);
}


void displayNumber(int size, int x, int y, float number)
{
  display.setTextSize(size);
  display.setTextColor(WHITE);
  display.setCursor(x, y);
  display.println(number);
}

// ==================================================
// RESISTANCE MEASUREMENT
// ==================================================

void calculateResistance()
{
  float Vref = 4.94;

  float Rref1 = 1000.0;
  float Rref2 = 10000.0;
  float Rref3 = 100000.0;

  float adc1 = 0;
  float adc2 = 0;
  float adc3 = 0;

  float v1 = 0;
  float v2 = 0;
  float v3 = 0;

  float r1 = 0;
  float r2 = 0;
  float r3 = 0;

  // Range 1
  pinMode(R1_PIN, OUTPUT);
  pinMode(R2_PIN, INPUT);
  pinMode(R3_PIN, INPUT);

  digitalWrite(R1_PIN, HIGH);

  for (int i = 0; i < 20; i++)
  {
    adc1 += analogRead(OHM_METER);
    delay(3);
  }

  adc1 /= 20.0;

  if (adc1 < 1022.9)
  {
    v1 = (adc1 * Vref) / 1024.0;
    r1 = (v1 * Rref1) / (Vref - v1);
  }

  // Range 2
  pinMode(R1_PIN, INPUT);
  pinMode(R2_PIN, OUTPUT);
  pinMode(R3_PIN, INPUT);

  digitalWrite(R2_PIN, HIGH);

  for (int i = 0; i < 20; i++)
  {
    adc2 += analogRead(OHM_METER);
    delay(3);
  }

  adc2 /= 20.0;

  if (adc2 < 1022.9)
  {
    v2 = (adc2 * Vref) / 1024.0;
    r2 = (v2 * Rref2) / (Vref - v2);
  }

  // Range 3
  pinMode(R1_PIN, INPUT);
  pinMode(R2_PIN, INPUT);
  pinMode(R3_PIN, OUTPUT);

  digitalWrite(R3_PIN, HIGH);

  for (int i = 0; i < 20; i++)
  {
    adc3 += analogRead(OHM_METER);
    delay(3);
  }

  adc3 /= 20.0;

  if (adc3 < 1022.9)
  {
    v3 = (adc3 * Vref) / 1024.0;
    r3 = (v3 * Rref3) / (Vref - v3);
  }

  // Select appropriate range

  if (r1 < 2000 && r1 > 0)
  {
    resistance = r1;
    kiloOhm = false;
  }
  else if (r2 < 101000 && r2 > 2000)
  {
    resistance = r2 / 1000.0;
    kiloOhm = true;
  }
  else if (r3 > 100000)
  {
    resistance = r3 / 1000.0;
    kiloOhm = true;
  }
  else
  {
    resistance = 0;
    kiloOhm = false;
  }
}

// ==================================================
// CAPACITANCE MEASUREMENT
// ==================================================

void calculateCapacitance()
{
  unsigned long startTime;
  unsigned long elapsedTime;

  float microFarads;
  float nanoFarads;

  float Rref = 10000.0;

  digitalWrite(CHARGE_PIN, HIGH);

  startTime = millis();

  while (analogRead(CAPACITANCE_METER) < 648)
  {
    // Wait until capacitor reaches threshold
  }

  elapsedTime = millis() - startTime;

  microFarads = ((float)elapsedTime / Rref) * 1000.0;

  if (microFarads > 1.0)
  {
    capacitance = microFarads;
    nanoFarad = false;
  }
  else
  {
    nanoFarads = microFarads * 1000.0;
    capacitance = nanoFarads;
    nanoFarad = true;
  }

  digitalWrite(CHARGE_PIN, LOW);

  // Discharge capacitor
  pinMode(DISCHARGE_PIN, OUTPUT);
  digitalWrite(DISCHARGE_PIN, LOW);

  while (analogRead(CAPACITANCE_METER) > 0)
  {
    // Wait until capacitor is discharged
  }

  pinMode(DISCHARGE_PIN, INPUT);
}

// ==================================================
// VOLTAGE MEASUREMENT
// ==================================================

void calculateVoltage()
{
  float R1 = 10000.0;
  float R2 = 4700.0;

  float Vref = 5.0;

  float resistorRatio;
  float adcValue = 0;
  float measuredVoltage;

  resistorRatio = R2 / (R1 + R2);

  // Average 20 readings
  for (int i = 0; i < 20; i++)
  {
    adcValue += analogRead(VOLT_METER);
    delay(3);
  }

  adcValue /= 20.0;

  measuredVoltage = (adcValue * Vref) / 1024.0;

  voltage = measuredVoltage / resistorRatio;
}

// ==================================================
// CURRENT MEASUREMENT
// ==================================================

void calculateCurrent()
{
  const int sensitivity = 185;

  const float Vref = 4.94;

  const float offsetVoltage = 2.47;

  int adcValue = 0;

  float measuredVoltage;
  float sensorVoltage;

  // Average 40 readings
  for (int i = 0; i < 40; i++)
  {
    adcValue += analogRead(AMMETER);
    delay(2);
  }

  adcValue /= 40;

  measuredVoltage = (adcValue * Vref) / 1024.0;

  sensorVoltage = measuredVoltage - offsetVoltage;

  sensorVoltage *= 1000.0;

  current = sensorVoltage / sensitivity;

  if (current < 1.0)
  {
    current *= 1000.0;
    milliAmp = true;
  }
  else
  {
    milliAmp = false;
  }
}

// ==================================================
// RESISTOR MODE
// ==================================================

void resistorMode()
{
  display.clearDisplay();

  displayText(1, 0, 0, "Resistance");

  displayText(2, 10, 10, "R=");

  displayNumber(2, 40, 10, resistance);

  if (kiloOhm)
  {
    displayText(1, 112, 16, "k");
  }
  else
  {
    displayText(1, 112, 16, "ohm");
  }

  display.display();

  calculateResistance();
}

// ==================================================
// VOLTAGE MODE
// ==================================================

void voltageMode()
{
  display.clearDisplay();

  displayText(1, 0, 0, "Voltage");

  displayText(2, 10, 10, "V=");

  displayNumber(2, 40, 10, voltage);

  displayText(1, 112, 16, "V");

  display.display();

  calculateVoltage();
}

// ==================================================
// CURRENT MODE
// ==================================================

void currentMode()
{
  display.clearDisplay();

  displayText(1, 0, 0, "Current");

  displayText(2, 10, 10, "I=");

  displayNumber(2, 40, 10, current);

  if (milliAmp)
  {
    displayText(1, 105, 16, "mA");
  }
  else
  {
    displayText(1, 112, 16, "A");
  }

  display.display();

  calculateCurrent();
}

// ==================================================
// CAPACITANCE MODE
// ==================================================

void capacitanceMode()
{
  display.clearDisplay();

  displayText(1, 0, 0, "Capacitance");

  displayText(2, 10, 10, "C=");

  displayNumber(2, 40, 10, capacitance);

  if (nanoFarad)
  {
    displayText(1, 105, 22, "nF");
  }
  else
  {
    displayText(1, 105, 22, "uF");
  }

  display.display();

  calculateCapacitance();
}

// ==================================================
// SETUP
// ==================================================

void setup()
{
  Serial.begin(9600);

  OLED_init();

  pinMode(RIGHT_BUTTON, INPUT_PULLUP);
  pinMode(SELECT_BUTTON, INPUT_PULLUP);

  pinMode(CHARGE_PIN, OUTPUT);

  digitalWrite(CHARGE_PIN, LOW);

  Serial.println("Digital Multimeter Started");
}

// ==================================================
// MAIN LOOP
// ==================================================

void loop()
{
  // ---------------- Navigation Button ----------------

  if (digitalRead(RIGHT_BUTTON) == LOW)
  {
    navigator++;

    while (digitalRead(RIGHT_BUTTON) == LOW)
    {
      // Wait for button release
    }

    delay(50);

    if (navigator > 3)
    {
      navigator = 0;
    }

    Serial.print("Selected Mode: ");
    Serial.println(navigator);
  }

  // ---------------- Select Button ----------------

  if (digitalRead(SELECT_BUTTON) == LOW)
  {
    isSelected = true;

    while (digitalRead(SELECT_BUTTON) == LOW)
    {
      // Wait for button release
    }

    delay(50);
  }

  // ---------------- Menu ----------------

  if (!isSelected)
  {
    display.clearDisplay();

    displayText(1, 0, 0, "SELECT MODE");

    if (navigator == 0)
    {
      displayText(2, 10, 12, "RESIST");
    }
    else if (navigator == 1)
    {
      displayText(2, 10, 12, "VOLT");
    }
    else if (navigator == 2)
    {
      displayText(2, 10, 12, "CURRENT");
    }
    else if (navigator == 3)
    {
      displayText(2, 10, 12, "CAP");
    }

    display.display();
  }

  // ---------------- Measurement ----------------

  if (isSelected)
  {
    if (navigator == 0)
    {
      resistorMode();
    }

    else if (navigator == 1)
    {
      voltageMode();
    }

    else if (navigator == 2)
    {
      currentMode();
    }

    else if (navigator == 3)
    {
      capacitanceMode();
    }

    // Press SELECT to exit measurement
    if (digitalRead(SELECT_BUTTON) == LOW)
    {
      isSelected = false;

      while (digitalRead(SELECT_BUTTON) == LOW)
      {
        // Wait for button release
      }

      delay(50);

      display.clearDisplay();
      display.display();
    }
  }
}
