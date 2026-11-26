---
layout: default
title: "4. Soudure et tests de la carte"
parent: journal
grand_parent: Journal de bord
nav_order: 4
---

# 4. Soudure et tests de la carte

**Sécurité** : la panne du fer dépasse 300 °C, on la repose toujours dans son support. On soude dans un endroit ventilé sans respirer les fumées, on porte des lunettes pour couper les pattes des composants traversants et on se lave les mains après la séance.

## Soudure de la carte
### Objectif
Assembler la carte finale de la console fournie par l'encadrant, avec son sachet de composants.

### Mise en place
**Avant de commencer**
- Comparer la carte à notre conception et repérer chaque emplacement (R1, C2, SW1…) sur le plan d'implantation.
- Trier les composants par référence. Une résistance CMS porte un marquage chiffré (102 = 1 kΩ, 472 = 4,7 kΩ), un condensateur n'en porte aucun.
- Fer réglé autour de 330 °C (étain sans plomb), panne étamée.

**Ordre de soudure : du plus bas au plus haut**, pour que la carte reste à plat jusqu'au bout :

| Ordre | Composants |
|-------|------------|
| 1 | Résistances et condensateurs CMS 1206 |
| 2 | LED témoin CMS (polarisée) |
| 3 | Boutons poussoirs CMS |
| 4 | Connecteur I2C |
| 5 | XIAO ESP32S3 sur ses barrettes |
| 6 | Joystick |
| 7 | Connecteur de l'écran (en dernier, l'écran se monte par-dessus) |

**Techniques**
- **CMS** : étamer une seule pastille, poser le composant à la brucelle en réchauffant la goutte, souder l'autre côté puis reprendre le premier. Une bonne soudure forme un congé lisse et brillant. Un composant dressé sur une pastille (effet pierre tombale) se reprend avec la tresse.
- **LED** : repérer la cathode sur la sérigraphie (trait, triangle ou K) et sur la LED (marquage vert, coin coupé). En cas de doute, utiliser le test de diode du multimètre.
- **Traversants** : souder une seule patte, vérifier que le composant est droit, puis souder les autres. Une bonne soudure a la forme d'un petit cône brillant. Couper les pattes à ras.
- **XIAO** : antenne débranchée, USB-C vers le bord ; ne pas surchauffer une même broche.
- **Joystick** : souder aussi les pattes de fixation, elles encaissent les efforts du pouce.

**Inspection à la loupe** avant toute mise sous tension : pas de pont d'étain (surtout sur les barrettes de la XIAO), pas de soudure sèche (terne, en boule), pas de patte oubliée, pas de résidu sur la carte.

### Résultat
> **À compléter :** photos recto et verso, soudures reprises (ce qui s'est passé, comment je l'ai trouvé).

## Tests et validation étape par étape
### Objectif
Vérifier chaque fonction de la carte avant de lancer le jeu. Un problème trouvé maintenant coûte cinq minutes ; le même découvert pendant le projet peut coûter une séance.

### Mise en place
**1. Hors tension (multimètre en mode continuité)**
- entre 3V3 et GND : aucun bip continu (un bip bref au premier contact est normal, les condensateurs se chargent) ;
- entre chaque bouton et sa broche de la XIAO : bip seulement bouton enfoncé ;
- entre chaque broche du connecteur écran et sa broche de la XIAO.

**2. Premier branchement (sans l'écran)** : LED témoin allumée, 3,3 V (± 0,1 V) entre 3V3 et GND, SDA et SCL à 3,3 V au repos (les résistances de tirage fonctionnent), rien ne chauffe. Au moindre doute, on débranche.

**3. Chaque fonction avec les programmes du TD2**, sans modification puisque le brochage est le même :

| Test | Programme | Attendu | Résultat |
|------|-----------|---------|----------|
| Flash | Blink | LED orange qui clignote | *à compléter* |
| SW1 (D5) | `justPressed()` | un message par appui | *à compléter* |
| SW2 (D6) | idem, `PIN_BUTTON = D6` | un message par appui | *à compléter* |
| Joystick | valeurs brutes puis normalisées | 0 au repos, -1 / +1 aux butées, appui détecté | *à compléter* |
| Écran | test d'affichage (monté carte débranchée) | texte et formes | *à compléter* |
| Tout ensemble | programme du point (sans buzzer) | le point suit le joystick | *à compléter* |

**Pannes courantes**

| Symptôme | À vérifier |
|----------|------------|
| Bip continu entre 3V3 et GND | pont d'étain autour des condensateurs et des barrettes |
| LED éteinte, 3,3 V présents | sens de la LED, soudure de sa résistance |
| SW2 ne réagit pas, SW1 si | soudure de la résistance série de D6 |
| Joystick bloqué à 0 ou 4095 | soudures du joystick, 3,3 V sur son VCC |
| Écran blanc | continuité SCK, MOSI, DC, RESET ; `SPI_MODE3` dans le code |
| Écran noir | broche LED du connecteur écran à 3,3 V |

### Résultat
> **À compléter :** tableau des tests rempli, défauts trouvés et corrections. Une carte dont on a documenté les défauts vaut mieux qu'une carte présentée comme parfaite.
