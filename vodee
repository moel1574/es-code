#include <TaskScheduler.h>
#include <Servo.h>
#include <Wire.h>
#include "rgb_lcd.h"

#define DEBUG

#if defined(ARDUINO_ARCH_SAM)
  #define MAX_VOLTAGE 3300
  #define WireMaster Wire1
  #define SETRES(x) analogReadResolution(x)
#elif defined(ARDUINO_ARCH_AVR)
  #define ADC_RESOLUTION 10
  #define MAX_VOLTAGE 5000
  #define WireMaster Wire
  #define SETRES(x) while(false){}
#else
  #error "Wrong Hardware"
#endif

#define timeOverflow 20000

Scheduler runner;

void tasterCallback();
void temperaturCallback();
void eingabeCallback();
void stellmotorCallback();

Task tasterTask(10 * TASK_MILLISECOND, TASK_FOREVER, &tasterCallback, &runner, true);
Task temperaturTask(1000 * TASK_MILLISECOND, TASK_FOREVER, &temperaturCallback, &runner, true);
Task eingabeTask(100 * TASK_MILLISECOND, TASK_FOREVER, &eingabeCallback, &runner, false);
Task stellmotorTask(3000 * TASK_MILLISECOND, TASK_FOREVER, &stellmotorCallback, &runner, true);

#define POTENTIOMETER_PIN A1
#define TASTER_PIN 2
#define TEMP_SENSOR_PIN A0

rgb_lcd lcd;
Servo myservo;

const int B = 4275;
const int R0 = 100000;

volatile bool tasterGedrueckt = false;
volatile unsigned long letzteInterruptZeit = 0;
const unsigned long debounceDelay = 50;

float temperatur = 0;
float sollTemperatur = 20;
bool inputAktiv = false;
unsigned long eingabeStartZeit = 0;

void setup() {
  Serial.begin(9600);

  myservo.attach(3);
  myservo.write(70);

  lcd.begin(16, 2, LCD_5x8DOTS, WireMaster);
  lcd.clear();

  pinMode(TASTER_PIN, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(TASTER_PIN), taster_ISR, FALLING);

  pinMode(POTENTIOMETER_PIN, INPUT);

  Serial.println("System gestartet.");

  runner.startNow();
}

void loop() {
  runner.execute();
}

void taster_ISR() {
  unsigned long aktuelleZeit = millis();
  if (aktuelleZeit - letzteInterruptZeit > debounceDelay) {
    tasterGedrueckt = true;
    letzteInterruptZeit = aktuelleZeit;
  }
}

void tasterCallback() {
  if (tasterGedrueckt) {
    tasterGedrueckt = false;

    if (inputAktiv) {
      inputAktiv = false;
      eingabeTask.disable();
      Serial.print("Solltemperatur bestaetigt: ");
      Serial.println(sollTemperatur);

      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Solltemp: ");
      lcd.print(sollTemperatur);
      lcd.print((char)223);
      lcd.print("C");
      lcd.setCursor(0, 1);
      lcd.print("Bestaetigt!");
    } else {
      inputAktiv = true;
      eingabeStartZeit = millis();
      eingabeTask.enable();
      Serial.println("Eingabe aktiviert.");

      lcd.clear();
    }
  }

  if (inputAktiv && (millis() - eingabeStartZeit > timeOverflow)) {
    inputAktiv = false;
    eingabeTask.disable();
    Serial.println("Eingabe durch Timeout beendet.");

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Timeout!");
    lcd.setCursor(0, 1);
    lcd.print("Eingabe beendet.");
  }
}

void temperaturCallback() {
  int sensorValue = analogRead(TEMP_SENSOR_PIN);
  float R = 1023.0 / sensorValue - 1.0;
  R = R0 * R;

  temperatur = 1.0 / (log(R / R0) / B + 1 / 298.15) - 273.15;

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Ist: ");
  lcd.print(temperatur);
  lcd.print((char)223);
  lcd.print("C");

  lcd.setCursor(0, 1);
  lcd.print("Soll: ");
  lcd.print(sollTemperatur);
  lcd.print((char)223);
  lcd.print("C");
}

void eingabeCallback() {
  if (inputAktiv) {
    int potentiometerValue = analogRead(POTENTIOMETER_PIN);
    sollTemperatur = 15.0 + ((20.0 * potentiometerValue) / 1023);
    Serial.print("Soll-Temp: ");
    Serial.println(sollTemperatur);

    lcd.setCursor(0, 1);
    lcd.print("Soll: ");
    lcd.print(sollTemperatur);
    lcd.print((char)223);
    lcd.print("C");
  }
}

void stellmotorCallback() {
  int pos;

  if (temperatur >= sollTemperatur + 5) {
    pos = 10;
  } else if (temperatur <= sollTemperatur - 5) {
    pos = 130;
  } else {
    pos = map(temperatur,
              sollTemperatur - 5,
              sollTemperatur + 5,
              130,
              10);
  }

  myservo.write(pos);
  Serial.print("Servo-Position: ");
  Serial.println(pos);
}