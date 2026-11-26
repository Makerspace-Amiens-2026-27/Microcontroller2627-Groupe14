---
layout: default
title: "6. Communication réseau"
parent: journal
grand_parent: Journal de bord
nav_order: 6
---

# 6. Communication réseau

## Wi-Fi, serveur web et WebSocket sur l'ESP32
### Objectif
La XIAO crée son propre réseau Wi-Fi. Un téléphone qui s'y connecte ouvre une page web servie par la carte, avec deux boutons qui allument et éteignent la LED. Dans l'autre sens, la carte envoie en continu la position du joystick, affichée en direct sur la page. C'est le squelette réseau du projet Snake.

### Mise en place
**Architecture** : le téléphone se connecte au point d'accès de la console (`192.168.4.1`), demande la page au **serveur HTTP** (port 80), puis ouvre un **WebSocket** (port 81) pour échanger des messages dans les deux sens.

⚠️ Il faut clipser l'**antenne Wi-Fi** sur la XIAO : sans elle, la portée tombe à quelques centimètres.

**Bibliothèques** : `WiFi.h` et `WebServer.h` font partie du core Arduino-ESP32. Le WebSocket vient de la bibliothèque de Links2004 :

```ini
lib_deps =
    links2004/WebSockets@^2.4.0
```

**Étapes**
1. **Point d'accès** : `WiFi.softAP(SSID, motDePasse)`. Le nom doit être unique dans la salle et le mot de passe faire au moins 8 caractères, sinon le point d'accès ne démarre pas.
2. **Page web** : `server.on("/", handleRoot)`. La page HTML est écrite dans une chaîne brute `R"rawliteral(...)rawliteral"`, et `server.handleClient()` est appelé à chaque tour de `loop()`.
3. **WebSocket** : `webSocket.begin()` et `webSocket.onEvent(...)`. Les événements (connexion, déconnexion, message) arrivent dans une fonction de rappel. Côté page, le JavaScript ouvre `ws://<adresse>:81/` et se reconnecte tout seul au bout d'une seconde si la liaison tombe.
4. **Navigateur → carte** : protocole texte `LED:1` / `LED:0`, tout autre message est ignoré. La fonction de rappel ne fait que mémoriser la commande, c'est `loop()` qui l'applique.
5. **Carte → navigateur** : `JOY:2048`, au plus 10 fois par seconde (cadencé par `millis()`) et seulement si la valeur a changé de plus de 40, pour ne pas saturer la liaison avec le bruit de l'ADC. `broadcastTXT()` envoie à tous les navigateurs connectés.

```cpp
case WStype_TEXT:
  // Message attendu : "LED:0" ou "LED:1", rien d'autre
  if (length == 5 && memcmp(payload, "LED:", 4) == 0) {
    if (payload[4] == '1') ledCommand = 1;
    if (payload[4] == '0') ledCommand = 0;
  }
  break;
```

**Intégration dans une game loop** : `server.handleClient()` et `webSocket.loop()` sont appelés à chaque tour, avant le test de cadence des images. Les messages reçus deviennent des entrées du jeu, comme les boutons, et la **console reste l'arbitre** : elle valide chaque demande avant de l'appliquer. `webSocket.connectedClients()` permet d'attendre un deuxième joueur ou de mettre le jeu en pause s'il se déconnecte.

**Pièges connus**
- Taper `http://192.168.4.1/` et pas `https://`, que la carte ne sait pas servir.
- Le réseau de la console n'a pas d'accès à Internet : certains téléphones repassent sur les données mobiles. Il faut choisir de rester connecté, ou couper les données le temps du test.

### Résultat
> **À compléter :** nom du réseau choisi, la page s'affiche-t-elle, la LED répond-elle, la valeur du joystick suit-elle en direct ? Test avec deux appareils connectés en même temps.
