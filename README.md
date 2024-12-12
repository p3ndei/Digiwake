#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <RTClib.h>

// Taille de l'écran OLED
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64

// Adresse I2C de l'écran OLED
#define OLED_ADDR 0x3C

// Initialisation de l'écran OLED
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire);

// Initialisation du module RTC
RTC_DS1307 rtcClock;

// Pins des boutons et du buzzer
#define BUTTON1_PIN 2 // Bouton de navigation dans le menu
#define BUTTON2_PIN 3 // Bouton de sélection
#define BUZZER_PIN 7  // Broche du buzzer

// Variables pour l'alarme
int wakeHour = 7;
int wakeMinute = 30;
bool alarmActive = true;
bool alarmTriggered = false;
bool alarmStopped = false;

// Variables pour le menu
int menuState = 0;
int menuOption = 0;

void setup() {
    // Initialisation série pour le débogage (facultatif)
    Serial.begin(9600);

    // Initialiser l'écran OLED
    if (!display.begin(SSD1306_I2C_ADDRESS, OLED_ADDR)) {
        Serial.println(F("Erreur d'initialisation de l'écran OLED!"));
        while (true);
    }

    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);
    display.println(F("Initialisation..."));
    display.display();

    // Initialiser le module RTC
    if (!rtcClock.begin()) {
        display.clearDisplay();
        display.println(F("Erreur RTC!"));
        display.display();
        while (true);
    }

    if (!rtcClock.isrunning()) {
        rtcClock.adjust(DateTime(F(__DATE__), F(__TIME__))); // Synchroniser avec l'heure de l'ordinateur
    }

    // Configurer les boutons et le buzzer
    pinMode(BUTTON1_PIN, INPUT_PULLDOWN);
    pinMode(BUTTON2_PIN, INPUT_PULLDOWN);
    pinMode(BUZZER_PIN, OUTPUT);

    delay(1000);
}

void loop() {
    static unsigned long lastUpdate = 0;
    unsigned long currentMillis = millis();

    // Mettre à jour l'écran toutes les secondes
    if (currentMillis - lastUpdate >= 1000) {
        lastUpdate = currentMillis;
        updateDisplay();

        // Vérifier si l'alarme doit sonner
        checkAlarm();
    }

    // Gérer les interactions avec les boutons
    handleMenu();
}

void updateDisplay() {
    DateTime now = rtcClock.now();

    // Effacer l'écran
    display.clearDisplay();

    // Afficher l'heure
    display.setTextSize(2);
    display.setCursor(0, 0);
    if (now.hour() < 10) display.print("0");
    display.print(now.hour());
    display.print(":");
    if (now.minute() < 10) display.print("0");
    display.print(now.minute());

    // Afficher la date
    display.setTextSize(1);
    display.setCursor(0, 32);
    display.print("Date: ");
    display.print(now.day());
    display.print("/");
    display.print(now.month());
    display.print("/");
    display.print(now.year());

    // Afficher l'état de l'alarme
    display.setCursor(0, 48);
    if (alarmActive) {
        display.print("Alarme: ");
        if (wakeHour < 10) display.print("0");
        display.print(wakeHour);
        display.print(":");
        if (wakeMinute < 10) display.print("0");
        display.print(wakeMinute);
    } else {
        display.print("Alarme desactivee");
    }

    display.display();
}

void handleMenu() {
    static bool button1Pressed = false;
    static bool button2Pressed = false;

    // Bouton 1 : Naviguer dans les options du menu
    if (digitalRead(BUTTON1_PIN) == HIGH && !button1Pressed) {
        menuOption = (menuOption + 1) % 3; // 3 options : heure, minute, activer/désactiver l'alarme
        button1Pressed = true;
        delay(200);
    } else if (digitalRead(BUTTON1_PIN) == LOW) {
        button1Pressed = false;
    }

    // Bouton 2 : Modifier l'option sélectionnée
    if (digitalRead(BUTTON2_PIN) == HIGH && !button2Pressed) {
        switch (menuOption) {
            case 0:
                wakeHour = (wakeHour + 1) % 24;
                break;
            case 1:
                wakeMinute = (wakeMinute + 1) % 60;
                break;
            case 2:
                alarmActive = !alarmActive;
                break;
        }
        button2Pressed = true;
        delay(200);
    } else if (digitalRead(BUTTON2_PIN) == LOW) {
        button2Pressed = false;
    }

    // Afficher le menu sur l'écran OLED
    display.clearDisplay();
    display.setTextSize(1);
    display.setCursor(0, 0);

    switch (menuOption) {
        case 0:
            display.println(F("Reglage heure"));
            break;
        case 1:
            display.println(F("Reglage minute"));
            break;
        case 2:
            display.println(alarmActive ? F("Alarme activee") : F("Alarme desactivee"));
            break;
    }

    display.display();
}

void checkAlarm() {
    if (!alarmActive || alarmStopped) return;

    DateTime now = rtcClock.now();
    if (now.hour() == wakeHour && now.minute() == wakeMinute) {
        alarmTriggered = true;
    }

    if (alarmTriggered) {
        // Activer le buzzer
        digitalWrite(BUZZER_PIN, HIGH);
        delay(500);
        digitalWrite(BUZZER_PIN, LOW);
        delay(500);
    }

    // Arrêter l'alarme si un bouton est pressé
    if (digitalRead(BUTTON1_PIN) == HIGH || digitalRead(BUTTON2_PIN) == HIGH) {
        alarmStopped = true;
        alarmTriggered = false;
    }
}
