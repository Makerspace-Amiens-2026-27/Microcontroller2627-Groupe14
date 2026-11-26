---
layout: default
title: "1. Installation de l'environnement"
parent: journal
grand_parent: Journal de bord
nav_order: 1
---

# 1. Installation de l'environnement

## Installation de VSCode et PlatformIO
### Objectif
Installer l'environnement de développement de l'atelier : l'éditeur **Visual Studio Code** et son extension **PlatformIO**, qui transforme VSCode en IDE complet pour microcontrôleurs (compilation, téléversement, moniteur série, gestion des bibliothèques).

PlatformIO est préféré à l'Arduino IDE parce qu'il gère mieux les projets et les bibliothèques (déclarées dans `platformio.ini`, donc versionnées avec le projet), compile plus vite et supporte des centaines de cartes, dont la XIAO ESP32S3.

### Mise en place
**VSCode (Windows)**
- Téléchargement de l'installateur sur [code.visualstudio.com](https://code.visualstudio.com/).
- Options cochées pendant l'installation : « Ouvrir avec Code » dans les menus contextuels, **ajout au PATH** (utile pour PlatformIO) et icône sur le bureau.

**PlatformIO**
- Dans VSCode : onglet Extensions (`Ctrl + Shift + X`), recherche de `platformio`, installation de **PlatformIO IDE**.
- La première installation est longue (plusieurs minutes) : PlatformIO télécharge Python si besoin, PlatformIO Core et les outils de compilation.
- Vérification : l'icône PlatformIO apparaît dans la barre de gauche, et **PIO Home** s'ouvre avec les sections New Project, Platforms, Libraries et Boards.

**Raccourcis utiles retenus**

| Raccourci | Action |
|-----------|--------|
| `Ctrl + Alt + B` | Build (compiler) |
| `Ctrl + Alt + U` | Upload (téléverser) |
| `Ctrl + Alt + S` | Moniteur série |

**Structure d'un projet PlatformIO**

```
mon_projet/
├── include/         # fichiers d'en-tête (.h), ex. config.h
├── lib/             # bibliothèques privées
├── src/main.cpp     # programme principal
├── test/            # tests unitaires
└── platformio.ini   # configuration : carte, framework, vitesse série, bibliothèques
```

Les bibliothèques se déclarent dans `platformio.ini` avec la clé `lib_deps` (format `auteur/Nom@^version`) et sont installées automatiquement à la compilation.

### Résultat
VSCode et PlatformIO sont installés et fonctionnels : PIO Home s'ouvre, et le projet `pong-esp32` compile et se téléverse sur la carte (voir la sous-partie suivante).

## Vérification de la toolchain et premier flash de la XIAO ESP32S3
### Objectif
Vérifier que toute la chaîne de développement fonctionne (compilation, téléversement, exécution) avec le programme le plus simple possible : faire clignoter la LED intégrée de la XIAO ESP32S3. C'est la seule sortie attendue du TD1.

### Mise en place
**Création du projet** dans PIO Home → *New Project* :
- Name : `pong-esp32`
- Board : Seeed Studio XIAO ESP32S3
- Framework : Arduino

La ligne `monitor_speed` est ajoutée à la main dans `platformio.ini` :

```ini
[env:seeed_xiao_esp32s3]
platform = espressif32
board = seeed_xiao_esp32s3
framework = arduino
monitor_speed = 115200
```

**Branchement** : le connecteur USB-C de la XIAO est relié directement à l'USB natif de l'ESP32-S3. Il n'y a pas de puce USB-série, donc **aucun driver à installer** : un nouveau port COM apparaît dans le Gestionnaire de périphériques. Il faut un câble USB-C qui transporte les données, pas seulement la charge.

**Programme Blink** :

```cpp
#include <Arduino.h>

void setup() {
  pinMode(LED_BUILTIN, OUTPUT);
}

void loop() {
  digitalWrite(LED_BUILTIN, LOW);   // LOW allume la LED de la XIAO
  delay(500);
  digitalWrite(LED_BUILTIN, HIGH);  // HIGH l'éteint
  delay(500);
}
```

À retenir :
- **`#include <Arduino.h>` est obligatoire** dans un `.cpp`. Contrairement à un `.ino` de l'Arduino IDE, il n'est pas ajouté automatiquement : sans lui, `pinMode` ou `Serial` sont inconnus et la compilation échoue.
- **LED en logique inversée** : la LED orange de la XIAO (GPIO21) est câblée vers le 3,3 V. Elle s'allume quand la broche est à `LOW` et s'éteint à `HIGH`. La LED rouge voisine indique la charge de la batterie et ne se pilote pas.
- **Si l'upload échoue** (`Failed to connect`, port introuvable) : débrancher la carte, maintenir **BOOT** (B), rebrancher, relâcher BOOT, relancer l'upload, puis appuyer sur **RESET** (R).

### Résultat
Le téléversement se termine par `[SUCCESS]` : le firmware (296 608 octets, 164 401 compressés) est écrit en 2,1 s puis vérifié par hash. La LED orange clignote à 1 Hz : la toolchain est validée.

![Téléversement réussi du programme Blink]({{ '/assets/mattheogaillard/flash-blink.png' | relative_url }})

## GitHub et organisation du dépôt du groupe
### Objectif
Travailler à plusieurs sur le même projet avec Git et GitHub, et tenir la documentation du projet (dont ce journal) dans le dépôt `Microcontroller2627-Groupe14` de l'organisation Makerspace-Amiens-2026-27.

### Mise en place
- Clonage du dépôt du groupe sur mon PC et ouverture dans VSCode, à côté du projet PlatformIO `pong-esp32`.
- Organisation du dépôt :

| Dossier | Contenu |
|---------|---------|
| `docs/` | site de documentation (Jekyll) : journal de bord, conception, fabrication, tests |
| `docs/journal/` | un journal par membre de l'équipe |
| `project/ecad/` | fichiers KiCad de la carte |
| `project/firmware/` | code de la console |
| `project/mcad/` | conception mécanique |
| `project/software/` | partie logicielle (page web du projet) |

- Workflow dans l'onglet **Source Control** de VSCode : ajouter les fichiers modifiés, faire un commit avec un message clair, puis **Sync / Push** pour les envoyer sur GitHub. Avant de commencer à travailler : **Pull** pour récupérer les changements des autres.
- Chaque page du journal commence par un en-tête (`title`, `parent`, `grand_parent`) qui la range dans le menu du site.

### Résultat
> **À compléter :** premier commit / push réussi, problèmes rencontrés (conflits, identifiants…).

## Port série et TelePlot
### Objectif
Visualiser en temps réel des données envoyées par la XIAO ESP32S3 via le port série, d'abord avec le moniteur série de PlatformIO, puis avec l'extension TelePlot.

### Mise en place
**La liaison série** utilise deux voies : TX (émission) et RX (réception), qu'il faut croiser pour relier deux cartes. Les deux côtés doivent utiliser la même vitesse, en bauds.

Sur la XIAO, `Serial` passe directement par l'USB natif de l'ESP32-S3 (option `ARDUINO_USB_CDC_ON_BOOT=1`, déjà activée par la configuration `seeed_xiao_esp32s3`) : **aucun fil TX/RX à croiser**. Conséquence : le port disparaît à chaque reset ou upload, d'où le `delay(1500)` au début de `setup()`, sinon les premiers messages sont perdus.

- Carte XIAO ESP32S3 reliée au PC en USB-C.
- Dans `platformio.ini` : `monitor_speed = 115200`.
- Extension TelePlot installée dans VSCode.

**Étape 1 : moniteur série simple.** Un premier programme envoie la valeur d'un sinus sous forme de texte toutes les 10 ms :

```cpp
void loop()
{
  i += 0.1;
  Serial.print("sin:");
  Serial.println(sin(i));
  delay(10);
}
```

Les valeurs défilent bien dans le moniteur série, mais elles sont difficiles à lire à cette vitesse.

![Moniteur série : valeurs du sinus]({{ '/assets/mattheogaillard/moniteur-serie-sin.png' | relative_url }})

**Étape 2 : TelePlot.** En ouvrant TelePlot sur le port COM3 à 115200 bauds avec un programme qui envoie `loop - i=...`, les lignes arrivent mais **aucune courbe ne s'affiche** : TelePlot ne trace que les lignes qui commencent par `>`. Les autres vont seulement dans le moniteur texte à droite.

![TelePlot : lignes reçues mais pas de courbe]({{ '/assets/mattheogaillard/teleplot-sans-prefixe.png' | relative_url }})

**Étape 3 : le bon format.** Pour tracer une courbe, chaque ligne doit avoir la forme `>nom:valeur`. Une petite fonction `teleplot()` le fait en un seul appel grâce à `Serial.printf()` :

```cpp
#include <Arduino.h>

float i = 0;

void teleplot(const char* label, float value)
{
  Serial.printf(">%s:%.3f\n", label, value);
}

void setup()
{
  Serial.begin(115200);
  delay(1500);   // USB natif : le port est rouvert après reset
}

void loop()
{
  i += 0.1;
  teleplot("sin", sin(i));
  teleplot("cos", cos(i));
  delay(10);
}
```

- `%s` insère le nom, `%.3f` la valeur avec trois décimales, `\n` termine la ligne.
- Le nom est un `const char*` et non un `String` : un `String` réserve de la mémoire à chaque appel, ce qui finit par fragmenter la mémoire sur un programme qui tourne longtemps.

### Résultat
Les courbes sin et cos s'affichent en temps réel, mises à jour toutes les 10 ms (environ 200 valeurs par seconde), et sont bien plus lisibles que le défilement du moniteur série.

![TelePlot : courbes sin et cos]({{ '/assets/mattheogaillard/teleplot-sin-cos.png' | relative_url }})

### Conclusion
TelePlot remplace avantageusement le moniteur série quand les données arrivent vite : il suffit de préfixer chaque ligne par `>` au format `>nom:valeur`. Les lignes sans `>` restent dans le moniteur texte, ce qui permet de mélanger courbes et messages de débogage.

Astuce : la commande **Upload and Monitor** de PlatformIO enchaîne le téléversement et la réouverture du moniteur dès que la carte redémarre.
