---
layout: default
title: "7. Projet Snake multijoueur"
parent: journal
grand_parent: Journal de bord
nav_order: 7
---

# 7. Projet Snake multijoueur

Le projet (24 h, en binôme) reprend la chaîne du Pong (lire → calculer → afficher, cadencée par la game loop et pilotée par une machine à états), avec une nouvelle mécanique (la grille, le corps qui grandit) et un second joueur dans un navigateur, relié à la console en Wi-Fi. Les deux joueurs ne jouent pas au même jeu : le multijoueur est **asymétrique**.

## Lancement : cahier des charges et répartition en binôme
### Objectif
Comprendre l'énoncé du projet, fixer les exigences et se répartir le travail.

### Mise en place
> **À compléter :** exigences de l'énoncé, rôle du joueur console et du joueur navigateur, répartition des tâches dans le binôme, planning.

### Résultat
> **À compléter**

## Découpage du code en modules (.h / .cpp)
### Objectif
Démarrer le projet avec un code organisé en modules, pour travailler à deux sans se gêner.

### Mise en place
Découpage proposé par le tutoriel du Pong et le tutoriel réseau :

| Module | Rôle |
|--------|------|
| `input.h/.cpp` | lecture des commandes (joystick, boutons) |
| `game.h/.cpp` | règles du jeu, sans aucun dessin |
| `renderer.h/.cpp` | dessin, sans aucune décision |
| `network.h/.cpp` | Wi-Fi, page web, WebSocket (`networkBegin()`, `networkPoll()`, `networkSendUpdates()`) |
| `main.cpp` | la game loop seulement |

Chaque `.h` déclare ce que le module offre, chaque `.cpp` explique comment il le fait, avec ses fonctions internes en `static`.

### Résultat
> **À compléter**

## La mécanique du Snake sur la console
### Objectif
Faire fonctionner le Snake sur l'écran de la console : déplacement sur une grille, corps qui grandit, collisions.

### Mise en place
> **À compléter**

### Résultat
> **À compléter**

## Le joueur navigateur (page web et WebSocket)
### Objectif
Permettre au second joueur d'agir sur la partie depuis la page web servie par la console.

### Mise en place
Point de départ : le squelette de la partie 6 (point d'accès, page HTTP, WebSocket dans les deux sens). Pistes du tutoriel : dessiner la grille dans la page avec un `<canvas>`, mettre la page dans `include/page.h`, passer au JSON (ArduinoJson / `JSON.parse()`) si le protocole se complique.

> **À compléter**

### Résultat
> **À compléter**

## Les états réseau et la synchronisation console ↔ navigateur
### Objectif
Ajouter à la machine à états du jeu les états liés au réseau (attente du second joueur, déconnexion) et garder la console et le navigateur synchronisés.

### Mise en place
> **À compléter**

### Résultat
> **À compléter**

## Tests et débogage
### Objectif

### Mise en place

### Résultat

## Bilan du projet
**Ce qui fonctionne**

**Ce qui reste à améliorer**

**Ce que j'ai appris**
