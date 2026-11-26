---
layout: default
title: "5. Le Pong sur la console"
parent: journal
grand_parent: Journal de bord
nav_order: 5
---

# 5. Le Pong sur la console

Premier jeu sur la carte soudée : un Pong à un joueur. La raquette en bas de l'écran suit le joystick, la balle rebondit sur les trois autres murs, et chaque balle manquée coûte une vie. On repart du programme de test « quatre modules ensemble » du TD2, qui contient déjà la game loop, la lecture du joystick, `justPressed()` et l'initialisation de l'écran.

## La game loop et l'affichage
### Objectif
Organiser les données du jeu, faire bouger la raquette, et afficher sans scintillement.

### Mise en place
**Les réglages** sont ajoutés à `include/config.h` : écran 240 × 320 en portrait, bandeau de score de 24 px en haut, une image toutes les 16 ms, raquette de 40 × 6 px, balle de 6 px à 3 px par image, 3 vies.

**Les données** sont regroupées dans des structures plutôt que dans des variables globales éparpillées :

```cpp
enum class GameState { Title, Playing, GameOver };

struct Paddle { float x; float y; };
struct Ball   { float x; float y; float vx; float vy; };   // vitesse en px par image

struct Game {
  GameState state;
  Paddle    paddle;
  Ball      ball;
  int       score;
  int       lives;
  uint32_t  serveTimeMs;   // instant où la balle sera lancée
};
```

Les positions sont des `float` (la balle peut avancer de 2,4 px par image) et désignent le coin supérieur gauche, comme pour `fillRect()`. Il ne faut pas confondre les **données** du jeu (positions, score : « où en est la partie ? ») et la **phase** du jeu (`game.state` : « quel écran afficher ? »).

**La raquette** avance proportionnellement à l'inclinaison du joystick (-1 à +1) et ne sort jamais de l'écran :

```cpp
static void movePaddle(Game& g, float joystickX) {
  g.paddle.x += joystickX * PADDLE_MAX_SPEED;
  g.paddle.x  = constrain(g.paddle.x, 0.0f, (float)(SCREEN_WIDTH - PADDLE_WIDTH));
}
```

Les fonctions de jeu reçoivent `Game&` pour la modifier, les fonctions de dessin `const Game&` : le compilateur interdit au dessin de modifier la partie.

**La game loop** garde les trois temps du TD2 :

```cpp
void loop() {
  static uint32_t lastFrameMs = 0;
  uint32_t nowMs = millis();
  if (nowMs - lastFrameMs < FRAME_TIME_MS) return;
  lastFrameMs = nowMs;

  float joystickX   = normalizeJoystick(analogRead(PIN_JOYSTICK_X), joystickCenterX);  // 1. Lire
  bool startPressed = justPressed(selectButton);
  gameUpdate(game, joystickX, startPressed, nowMs);                                    // 2. Calculer
  rendererDraw(game);                                                                  // 3. Afficher
}
```

**Afficher sans scintiller** : redessiner 240 × 320 pixels 60 fois par seconde en SPI ferait scintiller l'écran. Un écran complet n'est donc dessiné qu'au changement de phase. Pendant la partie, on n'efface et ne redessine que la raquette et la balle quand elles bougent, et le bandeau seulement quand le score ou les vies changent. L'affichage mémorise ce qui est à l'écran (`drawnPaddleX`, `drawnBallX`…). Quand on efface la balle alors qu'elle touche la raquette, on redessine la raquette pour ne pas y laisser un trou.

### Résultat
> **À compléter :** la raquette suit-elle le joystick ? L'affichage scintille-t-il ?

## Collisions et gestion des vies
### Objectif
Faire rebondir la balle sur les murs et la raquette de façon fiable, gérer le service, le score et les vies.

### Mise en place
- **Murs** : on inverse la composante de vitesse **et on replace la balle contre le mur**. Sans ce replacement, une balle qui dépasse de 2 px serait encore dehors à l'image suivante, sa vitesse s'inverserait une seconde fois et elle resterait collée.
- **Raquette** : test de chevauchement de deux rectangles (ils se touchent s'ils se chevauchent à la fois en x et en y), fait seulement si la balle descend, puis la balle est replacée au-dessus de la raquette.

```cpp
static bool ballTouchesPaddle(const Ball& b, const Paddle& p) {
  return b.x < p.x + PADDLE_WIDTH  && b.x + BALL_SIZE > p.x &&
         b.y < p.y + PADDLE_HEIGHT && b.y + BALL_SIZE > p.y;
}
```

- `moveBall()` renvoie `false` quand la balle sort par le bas : on perd une vie.
- **Service** : la balle repart du haut, en biais, après une pause d'une seconde. La pause ne doit **pas** être un `delay(1000)`, sinon la raquette ne pourrait plus bouger : on mémorise l'instant du service et la balle reste immobile tant qu'il n'est pas atteint.

```cpp
movePaddle(g, joystickX);
if (nowMs >= g.serveTimeMs && !moveBall(g)) {
  g.lives--;
  if (g.lives == 0) g.state = GameState::GameOver;
  else              serveBall(g, nowMs);
}
```

### Résultat
> **À compléter :** la balle reste-t-elle parfois collée ? Le score augmente-t-il à chaque renvoi ?

## La machine à états (menu / partie / fin)
### Objectif
Structurer le jeu en trois phases avec une machine à états finis (FSM).

### Mise en place
**Title** —(appui sur le joystick)→ **Playing** —(plus de vies)→ **GameOver** —(appui sur le joystick)→ **Title**

```cpp
static void gameUpdate(Game& g, float joystickX, bool startPressed, uint32_t nowMs) {
  switch (g.state) {
    case GameState::Title:
      if (startPressed) {
        g.score = 0;
        g.lives = START_LIVES;
        g.paddle.x = (SCREEN_WIDTH - PADDLE_WIDTH) / 2.0f;
        serveBall(g, nowMs);
        g.state = GameState::Playing;
      }
      break;
    case GameState::Playing:
      // déplacement, collisions, vies
      break;
    case GameState::GameOver:
      if (startPressed) g.state = GameState::Title;
      break;
  }
}
```

Les transitions se déclenchent sur un **appui** (`justPressed()`, vrai une seule fois), pas sur un bouton enfoncé : sinon un appui un peu long sur l'écran de fin renverrait à l'accueil puis relancerait aussitôt une partie.

`enum class` oblige à écrire `GameState::Title` : ces noms ne risquent pas d'entrer en conflit avec d'autres noms du programme.

### Résultat
> **À compléter :** le cycle complet fonctionne-t-il sans reflasher (accueil → partie → 3 balles perdues → fin → accueil) ? Améliorations ajoutées (deux joueurs avec SW1 / SW2, adversaire ordinateur, angle de rebond selon l'impact, découpage en modules input / game / renderer) ?
