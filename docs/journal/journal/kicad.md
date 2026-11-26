---
layout: default
title: "3. Conception de la carte sous KiCad"
parent: journal
grand_parent: Journal de bord
nav_order: 3
---

# 3. Conception de la carte sous KiCad

Au TD2, les modules vivaient sur une breadboard. Au TD3, on les réunit sur un circuit imprimé : plus de fils qui se débranchent, une console qui tient dans la main. La conception se fait en deux étapes, chacune validée par l'encadrant :

**Cahier des charges → Schéma (Eeschema) → Validation 1 → Routage (Pcbnew) → Validation 2 → Carte finale fournie → Soudure (TD4)**

Fabriquer un PCB prend une à trois semaines : une fois notre routage validé, l'encadrant nous remet une carte déjà fabriquée selon le même cahier des charges. Notre conception reste un livrable évalué et comparé à cette carte.

## Installation de KiCad et import des librairies (Seeed, SparkFun)
### Objectif
Installer KiCad (version 8 ou plus) et ajouter au projet les librairies des composants qui ne sont pas dans KiCad par défaut.

### Mise en place
**Installation (Windows)** : installateur `.exe` depuis [kicad.org](https://www.kicad.org/download/windows/), installation complète avec les bibliothèques de composants. Le gestionnaire de projets donne accès à Eeschema (schéma), Pcbnew (PCB) et GerbView.

**Deux types de librairies**
- **Symboles** (`.kicad_sym`) : la représentation du composant dans le schéma (Eeschema).
- **Empreintes** (dossier `.pretty`) : les pastilles et l'encombrement sur le PCB (Pcbnew).

**Librairies nécessaires pour la console**

| Composant | Librairie |
|-----------|-----------|
| XIAO ESP32S3 | Librairie KiCad de Seeed Studio (dossier série XIAO) |
| Joystick | Librairies KiCad de SparkFun (aussi dans le *Plugin and Content Manager*) |
| Résistances, condensateurs 1206, boutons | Librairies standard de KiCad ou librairie du MakerSpace |

**Import** : *Préférences → Gérer les librairies de symboles* (puis *d'empreintes*), onglet **Librairies spécifiques au projet**, bouton *Ajouter une librairie existante*. On sélectionne le fichier `.kicad_sym` pour les symboles et le **dossier** `.pretty` pour les empreintes.

On les ajoute **au projet** et non à l'installation : le projet reste autonome et s'ouvre sur un autre ordinateur sans rien installer.

**Vérification** : dans Eeschema, touche `A` puis recherche du composant ; il apparaît préfixé du nom de la librairie.

### Résultat
> **À compléter :** version de KiCad installée, librairies importées, problèmes rencontrés.

## Le schéma électronique sous Eeschema
### Objectif
Dessiner le schéma de la console en respectant le cahier des charges, puis le faire valider par l'encadrant (validation 1).

### Mise en place
**Cahier des charges : les fonctions**

| Fonction | Composant imposé | À concevoir |
|----------|------------------|-------------|
| Microcontrôleur | XIAO ESP32S3 sur barrettes | |
| Affichage | Écran TFT SPI sur connecteur 1 × 14 au pas de 2,54 mm | Câblage du connecteur |
| Joueur 1 | Joystick analogique SparkFun | |
| Joueur 2 | Deux boutons poussoirs CMS 6 × 6 mm | Protection de la broche D6 |
| Extension | Connecteur I2C 4 broches (3V3, SDA, SCL, GND) | Résistances de tirage |
| Témoin | LED allumée quand la carte est alimentée | Résistance série |
| Alimentation | USB-C de la XIAO, distribuée en 3,3 V | Découplage |

**Brochage imposé** (celui du TD2, sans le buzzer) :

| Broche XIAO | Signal |
|-------------|--------|
| D0 / D1 | Joystick, axes X / Y |
| D2 | Joystick, appui sur le stick |
| D3 / D4 | I2C : SDA / SCL |
| D5 / D6 | Boutons SW1 (gauche) / SW2 (droite), vers GND |
| D7 / D8 | Écran : RESET / DC |
| D9 / D10 | Écran : SCK / MOSI |

**Connecteur de l'écran (14 broches)** : 1 VCC → 3V3, 2 GND, 3 CS → GND, 4 RESET → D7, 5 DC → D8, 6 SDI → D10, 7 SCK → D9, 8 LED → 3V3, 9 SDO non connecté, 10 à 14 (tactile) non connectés.

**Méthode dans Eeschema**
- placer d'abord la XIAO et les connecteurs (`A`), relier les signaux avec des **étiquettes** (`L` : SCK, MOSI, SDA…) plutôt qu'avec de longs fils ;
- placer les symboles d'alimentation (+3V3, GND) et un `PWR_FLAG` là où l'alimentation entre sur la carte ;
- donner une valeur à chaque composant, **annoter** (R1, C1, SW1…) et remplir le cartouche ;
- lancer l'**ERC** et corriger jusqu'à 0 erreur ; mettre un marqueur de non-connexion (`Q`) sur les broches volontairement libres ;
- exporter le schéma en PDF dans le dépôt.

**Choix de conception à justifier (pistes de réflexion, à valider)**

| Question | Piste |
|----------|-------|
| D6 est aussi le TX de l'UART0 et émet au démarrage : que se passe-t-il si SW2 la relie à GND ? | Appuyer pendant que la broche est en sortie à `HIGH` crée un court-circuit vers GND. Une **résistance série** entre D6 et le bouton limite ce courant. |
| Pourquoi pas de résistance de tirage externe sur les boutons ? | Le code utilise `INPUT_PULLUP` : la résistance interne de l'ESP32-S3 suffit. |
| Résistances de tirage I2C | Sur SDA et SCL, vers 3V3 : **4,7 kΩ** (valeur de la carte finale). |
| Résistance de la LED témoin | R = (3,3 V − tension de la LED) / I. Pour une LED rouge (≈ 2 V) à 2 mA : environ 650 Ω, donc une valeur normalisée proche (680 Ω ou 1 kΩ). |
| Découplage | Condensateurs (100 nF, voire 10 µF) entre 3V3 et GND, **au plus près** des broches d'alimentation. |
| Pourquoi alimenter le joystick en 3V3 ? | Ses sorties vont sur l'ADC de l'ESP32-S3, qui ne supporte pas plus de 3,3 V. |

**Liste de contrôle de la validation 1** : ERC sans erreur, brochage conforme, connecteur écran vérifié sur la sérigraphie du module, choix de conception justifiés, valeur et empreinte (1206) pour chaque composant, cartouche rempli, PDF dans le dépôt.

### Résultat
> **À compléter :** capture du schéma, résultat de l'ERC, valeurs choisies (résistances, condensateurs), remarques de l'encadrant à la validation 1.

## Le routage du PCB sous Pcbnew
### Objectif
Transformer le schéma validé en circuit imprimé : placer les composants, router les pistes, vérifier avec le DRC, puis faire valider (validation 2).

### Mise en place
**Contraintes de fabrication** (limites d'un fabricant comme AISLER, dont on peut importer les règles DRC) :

| Contrainte | Valeur |
|------------|--------|
| Dimensions | 90 × 50 mm |
| Couches | 2 (cuivre dessus et dessous) |
| Fixation | 4 trous M3, un dans chaque coin |
| CMS | Boîtier 1206 minimum (soudure au fer) |
| Pistes / isolement | 0,2 mm minimum, plus large pour l'alimentation |
| Vias | Diamètre 0,7 mm, perçage 0,3 mm |

**Méthode dans Pcbnew**
1. Mettre à jour le PCB depuis le schéma (`F8`) : les empreintes arrivent en vrac, reliées par le **chevelu**.
2. Dessiner le contour 90 × 50 mm sur la couche **Edge.Cuts** et placer les 4 trous M3.
3. Régler les règles de conception (*Fichier → Configuration du circuit imprimé*).
4. **Placer avant de router**, en pensant à l'usage : la console se tient à deux mains, joystick sous un pouce, boutons sous l'autre, écran visible entre les deux, USB-C accessible depuis un bord. Vérifier dans la vue 3D (`Alt + 3`) que l'écran ne recouvre ni le joystick ni les boutons.
5. Router (`X`), en mode 45° : condensateurs de découplage au plus près, pistes d'alimentation plus larges, bus SPI de l'écran court et direct. Changement de couche avec un via (`V`).
6. Ajouter un **plan de masse** (zone de cuivre reliée à GND) sur la face arrière, voire les deux, et le remplir (`B`).
7. Sérigraphie : SW1, SW2, broche 1 des connecteurs, nos noms, révision.
8. Lancer le **DRC** jusqu'à 0 erreur et plus aucune connexion en attente.

### Résultat
> **À compléter :** captures 2D et 3D, résultat du DRC, remarques de l'encadrant à la validation 2.

## La carte de la console : du cahier des charges au routage validé
### Objectif
Faire le bilan de la conception : ce qui a été validé, ce qui a été corrigé, et les différences avec la carte finale fournie par l'encadrant.

### Mise en place
**Liste de contrôle de la validation 2** : DRC sans erreur, carte de 90 × 50 mm avec un trou M3 dans chaque coin, placement ergonomique, découplage au plus près, pistes d'alimentation plus larges, plan de masse, sérigraphie complète, fichiers KiCad et captures dans le dépôt.

### Résultat
> **À compléter :** corrections demandées entre les deux validations, comparaison de notre placement avec celui de la carte finale, ce que je referais autrement.
