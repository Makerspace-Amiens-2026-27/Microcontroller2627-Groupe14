---
layout: default
title: "2. Les modules de la mallette"
parent: journal
grand_parent: Journal de bord
nav_order: 2
---

# 2. Les modules de la mallette (sur breadboard)

La mallette contient les quatre modules de la future console : un bouton, un joystick, un buzzer et un écran. Avant de dessiner la carte qui les réunira, on teste chacun séparément sur breadboard, toujours avec la même démarche :

1. identifier ses broches et sa tension d'alimentation ;
2. câbler une seule chose à la fois ;
3. le faire fonctionner avec le programme le plus simple possible ;
4. valider au moniteur série (ou à l'oreille, à l'œil) avant de passer au suivant.

**Règle de sécurité : tout en 3,3 V.** Tous les modules sont alimentés sur la broche 3V3 de la XIAO, jamais sur 5V : les entrées de l'ESP32-S3 ne supportent pas plus de 3,3 V.

## Le brochage de la console (`config.h`)
### Objectif
Regrouper dans un seul fichier toutes les broches utilisées par les modules, pour que chaque programme de test utilise le même câblage. On utilise dès maintenant les broches de la future carte : un programme écrit sur breadboard fonctionnera sans modification sur la carte soudée.

### Mise en place

| Module | Broche du module | Broche XIAO |
|--------|------------------|-------------|
| Bouton | Signal (S) | D5 |
| Joystick | VRx / VRy | D0 / D1 (ADC1) |
| Joystick | SW (appui sur le stick) | D2 |
| Buzzer | Signal (S) | D3 (seulement sur breadboard) |
| Écran | SCK (sérigraphié SCL) | D9 |
| Écran | SDI / MOSI (sérigraphié SDA) | D10 |
| Écran | DC / RESET | D8 / D7 |
| Écran | CS | GND |
| Écran | LED (rétroéclairage) | 3V3 |
| Tous | VCC / GND | 3V3 / GND |

![Brochage de la XIAO ESP32S3]({{ '/assets/mattheogaillard/brochage-xiao.png' | relative_url }})

```cpp
// include/config.h : brochage de la console
#pragma once
#include <Arduino.h>

constexpr int PIN_BUTTON          = D5;
constexpr int PIN_JOYSTICK_X      = D0;
constexpr int PIN_JOYSTICK_Y      = D1;
constexpr int PIN_JOYSTICK_SELECT = D2;
constexpr int PIN_BUZZER          = D3;   // seulement sur breadboard
constexpr int PIN_TFT_RESET       = D7;
constexpr int PIN_TFT_DC          = D8;
constexpr int PIN_TFT_SCK         = D9;
constexpr int PIN_TFT_MOSI        = D10;
constexpr int PIN_TFT_CS          = -1;   // CS relié à GND : pas de broche
```

`constexpr int` déclare une constante typée : comme un `#define`, le compilateur la remplace par sa valeur, mais il vérifie en plus son type.

### Résultat
Tous les tests de la séance incluent `config.h` : changer une broche ne demande qu'une modification à un seul endroit.

## Le bouton poussoir (GPIO en entrée)
### Objectif
Compter les appuis sur le bouton, en ne comptant qu'**un seul** appui même si le contact rebondit.

### Mise en place
Câblage : VCC sur 3V3, GND sur GND, signal S sur D5.

**Étape 1 : lire le niveau.** Selon le module, le signal vaut `LOW` ou `HIGH` quand on appuie. Plutôt que de deviner, on affiche le niveau lu, bouton relâché puis enfoncé :

```cpp
#include "config.h"

void setup() {
  Serial.begin(115200);
  pinMode(PIN_BUTTON, INPUT_PULLUP);
}

void loop() {
  Serial.println(digitalRead(PIN_BUTTON) == HIGH ? "HIGH" : "LOW");
  delay(200);   // programme de test : on ralentit l'affichage
}
```

`INPUT_PULLUP` active la résistance de tirage interne de l'ESP32-S3 : elle ne gêne pas un module qui a déjà la sienne, et elle évite une entrée flottante s'il n'en a pas.

> **À compléter :** niveau lu bouton relâché / enfoncé.

**Étape 2 : détecter un appui une seule fois.** Pour un jeu, on veut savoir si le bouton *vient* d'être enfoncé (détection de front). Deux pièges : tant que le doigt reste appuyé, `loop()` voit le bouton enfoncé des milliers de fois, et le contact mécanique rebondit pendant quelques millisecondes.

- Broche en `INPUT_PULLUP` : la résistance de tirage interne maintient la broche à `HIGH`, et l'appui la met à `LOW` (`PRESSED_LEVEL = LOW`).
- **Anti-rebond (debounce)** : un contact mécanique oscille quelques millisecondes à chaque appui. Le changement n'est validé que si le niveau reste stable pendant 30 ms.
- La fonction `justPressed()` renvoie `true` uniquement au moment de l'appui, pas pendant que le bouton reste enfoncé ni au relâchement.

```cpp
#include "config.h"

constexpr int PRESSED_LEVEL = LOW;
constexpr unsigned long DEBOUNCE_MS = 30;

struct Button {
  int pin;
  int stableLevel;
  int lastRead;
  unsigned long lastChangeMs;
};

Button button = {PIN_BUTTON, !PRESSED_LEVEL, !PRESSED_LEVEL, 0};

bool justPressed(Button& b) {
  int level = digitalRead(b.pin);
  if (level != b.lastRead) {
    b.lastRead = level;
    b.lastChangeMs = millis();
  }
  if (millis() - b.lastChangeMs > DEBOUNCE_MS && level != b.stableLevel) {
    b.stableLevel = level;            // stable depuis 30 ms : changement confirmé
    return level == PRESSED_LEVEL;    // vrai sur l'appui, faux sur le relâchement
  }
  return false;
}

void setup() {
  Serial.begin(115200);
  pinMode(PIN_BUTTON, INPUT_PULLUP);
}

void loop() {
  static int count = 0;
  if (justPressed(button)) {
    count++;
    Serial.printf("Appui n°%d\n", count);
  }
}
```

### Résultat
Chaque appui affiche `Appui n°1`, `Appui n°2`… dans le moniteur série, un seul message par appui.

## Le joystick analogique (ADC)
### Objectif
Lire les deux axes du joystick avec l'ADC et les traduire en une direction : HAUT, BAS, GAUCHE, DROITE ou CENTRE.

### Mise en place
- Chaque axe est un potentiomètre lu par `analogRead()`. L'ADC de l'ESP32-S3 est sur 12 bits : valeur de 0 à 4095, environ 2048 au repos.
- Seuils : en dessous de 1500 ou au-dessus de 2500, on considère que le joystick est poussé. Entre les deux, c'est une zone morte (CENTRE).
- Lecture toutes les 50 ms avec `millis()` (sans `delay()` bloquant), et affichage seulement quand la direction **change**.
- Le bouton du joystick (SW) est lu en `INPUT_PULLUP`, comme le bouton poussoir.

**Problème rencontré :** la première version (Max) utilisait les broches `34` et `35`. Ce sont des numéros de GPIO de l'ESP32 classique, qui ne sont pas sortis sur la XIAO ESP32S3. La version corrigée utilise `A0` et `A1`, conformes à `config.h`.

```cpp
#include <Arduino.h>

#define PIN_JOYSTICK_X  A0
#define PIN_JOYSTICK_Y  A1
#define PIN_JOYSTICK_SW D2

#define CENTRE 0
#define HAUT   1
#define BAS    2
#define GAUCHE 3
#define DROITE 4

unsigned long dernierTemps = 0;
const unsigned long PERIODE = 50;
int ancienneDirection = CENTRE;

int lireDirection() {
  int x = analogRead(PIN_JOYSTICK_X);
  int y = analogRead(PIN_JOYSTICK_Y);
  if (x < 1500) return GAUCHE;
  if (x > 2500) return DROITE;
  if (y < 1500) return BAS;
  if (y > 2500) return HAUT;
  return CENTRE;
}

void afficherDirection(int direction) {
  if (direction == HAUT)        Serial.println("HAUT");
  else if (direction == BAS)    Serial.println("BAS");
  else if (direction == GAUCHE) Serial.println("GAUCHE");
  else if (direction == DROITE) Serial.println("DROITE");
  else                          Serial.println("CENTRE");
}

void setup() {
  Serial.begin(115200);
  pinMode(PIN_JOYSTICK_SW, INPUT_PULLUP);
  Serial.println("TEST JOYSTICK");
}

void loop() {
  unsigned long tempsActuel = millis();
  if (tempsActuel - dernierTemps >= PERIODE) {
    dernierTemps = tempsActuel;

    int direction = lireDirection();
    if (direction != ancienneDirection) {
      afficherDirection(direction);
      ancienneDirection = direction;
    }

    if (digitalRead(PIN_JOYSTICK_SW) == LOW) {
      Serial.println("BOUTON APPUYE");
    }
  }
}
```

### Résultat
Chaque mouvement du joystick affiche la bonne direction une seule fois. Le bouton du joystick affiche `BOUTON APPUYE` toutes les 50 ms tant qu'il reste enfoncé : on pourra réutiliser `justPressed()` pour ne détecter que l'appui.

### Pour aller plus loin : calibrer et normaliser
Notre version donne seulement une direction. Pour le jeu (vitesse de la raquette du Pong), le tutoriel demande une valeur **entre -1 et +1**, qui vaut 0 au repos :

- **calibrer** : au démarrage, mesurer le vrai centre (moyenne de 16 lectures, stick relâché) au lieu de supposer 2048. Au repos, la valeur est proche de 2048 mais presque jamais exacte (souvent entre 1800 et 2100) ;
- **zone morte** : un écart de moins de 300 autour du centre est du bruit, on renvoie 0 ;
- **course asymétrique** : on divise par la course disponible du côté où se trouve le stick, pour que chaque butée donne bien -1 ou +1.

```cpp
constexpr int JOYSTICK_DEADZONE = 300;
int joystickCenterX = 2048;

void calibrateJoystick() {
  long sum = 0;
  for (int i = 0; i < 16; i++) {
    sum += analogRead(PIN_JOYSTICK_X);
    delay(2);
  }
  joystickCenterX = sum / 16;
}

// Convertit une lecture brute (0..4095) en valeur entre -1.0 et +1.0
float normalizeJoystick(int raw, int center) {
  int offset = raw - center;
  if (abs(offset) < JOYSTICK_DEADZONE) {
    return 0.0f;
  }
  int range = (offset > 0) ? (4095 - center) : center;
  return constrain((float)offset / range, -1.0f, 1.0f);
}
```

> **À compléter :** valeur de x et y au repos mesurée, et vérification 0.00 au repos / -1.00 et +1.00 aux butées.

## Le buzzer (PWM)
### Objectif
Faire sonner le buzzer et comprendre la différence entre un buzzer **passif** et un buzzer **actif**.

### Mise en place
- **Buzzer passif** : il faut lui envoyer un signal carré à la fréquence de la note. `tone(broche, fréquence, durée)` génère ce signal en PWM.
- **Buzzer actif** : il contient son propre oscillateur, il suffit de le mettre à `HIGH` pour qu'il bipe.
- Le programme joue d'abord une gamme do-ré-mi-fa-sol, puis un bip d'une demi-seconde : selon ce qu'on entend, on sait quel type de buzzer est monté.

```cpp
#include "config.h"

void setup() {
  pinMode(PIN_BUZZER, OUTPUT);

  // Test 1 : buzzer passif, une gamme do-ré-mi-fa-sol
  const int notes[] = {262, 294, 330, 349, 392};   // fréquences en Hz
  for (int note : notes) {
    tone(PIN_BUZZER, note, 200);
    delay(250);   // test de démarrage : attendre la fin de la note
  }
  delay(1000);

  // Test 2 : buzzer actif, un bip d'une demi-seconde
  digitalWrite(PIN_BUZZER, HIGH);
  delay(500);
  digitalWrite(PIN_BUZZER, LOW);
}

void loop() {}
```

Les `delay()` sont acceptables ici, dans un programme de test qui ne fait rien d'autre. Dans un jeu, une note se lance avec `tone()` et on ne l'attend pas.

### Résultat
> **À compléter :** gamme entendue (buzzer passif) ou seulement des clics puis un bip (buzzer actif) ? Type de buzzer de notre mallette : …

## L'écran TFT ST7789 (bus SPI)
### Objectif
Afficher du texte sur l'écran TFT couleur 240 × 320, piloté en SPI avec les bibliothèques Adafruit GFX et Adafruit ST7789.

### Mise en place
- Câblage : SCK sur D9, MOSI (SDA) sur D10, DC sur D8, RST sur D7. La broche CS est reliée à GND, donc `TFT_CS = -1`.
- Le bus SPI est démarré sur ces broches personnalisées avec `SPI.begin(SCK, MISO, MOSI, SS)`. MISO n'est pas utilisée : l'écran ne fait que recevoir.
- **Point important :** sans broche CS, l'écran ne fonctionne qu'en `SPI_MODE3`. Il faut le préciser à `tft.init()`.
- Rotation 1 : affichage en paysage pour ce test. Le tutoriel et le Pong utilisent le **portrait** (rotation 2, 240 px de large et 320 de haut, origine (0, 0) en haut à gauche).
- Bibliothèque déclarée dans `platformio.ini` :

```ini
lib_deps =
    adafruit/Adafruit ST7735 and ST7789 Library@^1.11.0
```

```cpp
#include <Arduino.h>
#include <SPI.h>
#include <Adafruit_GFX.h>
#include <Adafruit_ST7789.h>

constexpr int TFT_SCK  = D9;    // SCL / SCK
constexpr int TFT_MOSI = D10;   // SDA / MOSI
constexpr int TFT_DC   = D8;    // DC / RS
constexpr int TFT_RST  = D7;    // RES / RST
constexpr int TFT_CS   = -1;    // -1 si pas de broche CS (ou CS reliée à GND)

constexpr int TFT_WIDTH    = 240;
constexpr int TFT_HEIGHT   = 320;
constexpr int TFT_ROTATION = 1;   // 0 = portrait, 1 = paysage

Adafruit_ST7789 tft(&SPI, TFT_CS, TFT_DC, TFT_RST);

void drawText(int x, int y, const char* text, uint16_t color, uint8_t size) {
  tft.setTextColor(color);
  tft.setTextSize(size);
  tft.setCursor(x, y);
  tft.print(text);
}

void setup() {
  Serial.begin(115200);
  delay(500);
  Serial.println("Demarrage...");

  // SPI sur broches personnalisées : SCK, MISO (inutilisée), MOSI, SS
  SPI.begin(TFT_SCK, -1, TFT_MOSI, -1);

  // Sans broche CS, l'écran a besoin du SPI_MODE3
  if (TFT_CS < 0) {
    tft.init(TFT_WIDTH, TFT_HEIGHT, SPI_MODE3);
  } else {
    tft.init(TFT_WIDTH, TFT_HEIGHT);
  }

  tft.setRotation(TFT_ROTATION);
  tft.fillScreen(ST77XX_BLACK);

  drawText(20, 40,  "GAME BOY",      ST77XX_GREEN, 3);
  drawText(20, 100, "Test ecran OK", ST77XX_WHITE, 2);

  Serial.printf("Ecran pret : %d x %d\n", tft.width(), tft.height());
}

void loop() { }
```

### Résultat
L'écran affiche « GAME BOY » et « Test ecran OK » : le câblage SPI et l'initialisation sont validés.

**Problème restant :** les couleurs sont inversées. Le fond est blanc au lieu de noir et le titre violet au lieu de vert. La bibliothèque active l'inversion des couleurs par défaut pour le ST7789, et ce modèle d'écran n'en a pas besoin. Piste à tester : ajouter `tft.invertDisplay(false);` après `tft.init()`.

![Écran TFT : GAME BOY, Test écran OK]({{ '/assets/mattheogaillard/tft-game-boy.png' | relative_url }})

## Les quatre modules ensemble
### Objectif
Dernier test du TD2, qui servira aussi de programme de test de la carte soudée : un point blanc se déplace à l'écran avec le joystick, et le buzzer bipe à chaque appui sur le bouton.

### Mise en place
Le programme réutilise tout ce qui précède : `justPressed()`, la calibration et la normalisation du joystick, l'initialisation de l'écran. La boucle ne contient **aucun `delay()`** : une image toutes les 16 ms, cadencée par `millis()`. C'est la *game loop* (lire → calculer → afficher) que le Pong reprendra.

```cpp
constexpr uint32_t FRAME_TIME_MS = 16;

float dotX = 120, dotY = 160;     // position calculée
int drawnX = 120, drawnY = 160;   // position affichée

void loop() {
  static uint32_t lastFrameMs = 0;
  uint32_t nowMs = millis();
  if (nowMs - lastFrameMs < FRAME_TIME_MS) {
    return;
  }
  lastFrameMs = nowMs;

  // 1. Lire
  float x = normalizeJoystick(analogRead(PIN_JOYSTICK_X), joystickCenterX);
  bool beep = justPressed(button);

  // 2. Calculer
  dotX = constrain(dotX + x * 4.0f, 0.0f, 235.0f);
  if (beep) {
    tone(PIN_BUZZER, 880, 50);
  }

  // 3. Afficher : effacer l'ancien point, dessiner le nouveau
  if ((int)dotX != drawnX) {
    tft.fillRect(drawnX, drawnY, 5, 5, ST77XX_BLACK);
    drawnX = (int)dotX;
    tft.fillRect(drawnX, drawnY, 5, 5, ST77XX_WHITE);
  }
}
```

On n'efface et ne redessine que le point quand il bouge, au lieu de tout l'écran : c'est ce qui évite le scintillement.

### Résultat
> **À compléter :** le point suit-il le joystick sans laisser de traînée ? Le bip ralentit-il le point ?

## Bilan du TD2 : données d'entrée pour la carte

| Module | Alimentation | Ce qu'il renvoie / attend | Programme minimal |
|--------|--------------|---------------------------|-------------------|
| Bouton | 3V3 | `LOW` à l'appui, avec `INPUT_PULLUP` | `justPressed()` avec anti-rebond de 30 ms |
| Joystick | 3V3 (même si sérigraphié +5V) | 0 à 4095, centre ≈ *(à compléter)* ; SW à `LOW` à l'appui | `analogRead()` + normalisation -1 / +1 |
| Buzzer | 3V3 | *(actif / passif : à compléter)* | `tone()` ou `digitalWrite()` |
| Écran TFT | 3V3 (VCC et LED) | SPI, CS à GND donc `SPI_MODE3` | `tft.init(240, 320, SPI_MODE3)` |

## L'écran OLED SSD1306 (bus I2C)
> Ce module ne fait pas partie de la mallette du TD2 : il peut se brancher sur le connecteur I2C de la carte de la console.

### Objectif
Afficher du texte et des formes sur un petit écran OLED monochrome 128 × 64, piloté en **I2C**.

### Mise en place
**Le bus I2C** n'utilise que deux fils, partagés par tous les périphériques : **SDA** (données, dans les deux sens) et **SCL** (horloge, générée par l'ESP32). Chaque périphérique a une adresse : 0x3C par défaut pour le SSD1306, parfois 0x3D.

- Sur la carte de la console, l'I2C sort sur un connecteur 4 broches (3V3, SDA, SCL, GND), avec **SDA sur D3** et **SCL sur D4**, et des résistances de tirage de 4,7 kΩ déjà soudées. Ce ne sont pas les broches I2C par défaut de la XIAO : il faut écrire `Wire.begin(D3, D4);`.
- Le SSD1306 accepte 3,0 à 5,5 V et ses entrées sont compatibles 3,3 V : connexion directe, sans adaptateur de niveau.
- Bibliothèques dans `platformio.ini` :

```ini
lib_deps =
    adafruit/Adafruit SSD1306@^2.5.7
    adafruit/Adafruit GFX Library@^1.11.3
```

```cpp
#include <Arduino.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

constexpr int SCREEN_WIDTH   = 128;
constexpr int SCREEN_HEIGHT  = 64;
constexpr int OLED_RESET     = -1;     // pas de broche RESET
constexpr int SCREEN_ADDRESS = 0x3C;

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

void setup() {
  Serial.begin(115200);
  delay(500);
  Wire.begin(D3, D4);   // SDA, SCL de la carte de la console

  if (!display.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS)) {
    Serial.println("Erreur : ecran SSD1306 non detecte");
    while (true) delay(1000);
  }

  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println("XIAO + SSD1306");
  display.setTextSize(2);
  display.println("Hello!");
  display.display();    // envoie le tampon vers l'ecran
}

void loop() {}
```

**À retenir :** l'affichage est **bufferisé**. Les fonctions de dessin modifient un tampon de 1024 octets en RAM, et rien n'apparaît tant qu'on n'appelle pas `display.display()`, qui envoie 1 Ko en I2C (environ 8 ms à 400 kHz).

**Si rien ne s'affiche** : lancer un scanner I2C (boucle sur les adresses 1 à 126 avec `Wire.beginTransmission()`) pour vérifier que l'écran répond en 0x3C ou 0x3D, puis contrôler le câblage et l'alimentation.

### Résultat
> **À compléter :** adresse détectée, texte affiché, photo de l'écran.
