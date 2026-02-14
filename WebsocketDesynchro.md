## 📋 Description Rapide
Il est possible de contourner le blocage de sécurité empêchant un joueur de rejoindre plusieurs parties simultanément.

En initiant une nouvelle connexion WebSocket sur une partie où le joueur est déjà présent, le système semble réinitialiser le statut du joueur à **"pas en partie"** tout en maintenant sa session de jeu active. Cela permet ensuite de rejoindre d'autres parties en parallèle.

## 🛠 Étapes de reproduction

1. Rejoindre la **Partie A** (Établissement de la WebSocket n°1).
2. Depuis le même compte, tenter de rejoindre à nouveau la **Partie A** (ex: via duplication de l'onglet).
3. **Observation :** La WebSocket n°2 prend le relais sur la Partie A et la WebSocket n°1 est déconnectée par le serveur.
4. Attendre un délai quelconque (le bug persiste indéfiniment, invalidant l'hypothèse de lag de réplication).
5. Tenter de rejoindre une **Partie B** totalement différente.

**Résultat :** Le jeu autorise l'accès à la Partie B sans erreur de type *"Vous êtes déjà en jeu"* alors que le joueur est encore actif sur la partie A.

## 🆚 Comportement Attendu vs Actuel

| Type | Description |
| :--- | :--- |
| **Attendu** | Le système doit détecter que l'ID du joueur est toujours associé à la Partie A et bloquer toute nouvelle connexion à une Partie B tant que la Partie A n'est pas quittée. |
| **Actuel** | Le système considère le joueur comme `Libre` après la re-connexion (chevauchement des websockets) à la même partie, permettant le multi-partie. |

---

## 🕵️‍♂️ Analyse Technique & Investigation

Suite à l'étude de l'architecture (Node.js + WebSockets + Redis/PostgreSQL), le problème a été identifié comme une **Race Condition critique** lors du nettoyage de session.

### ❌ Hypothèse invalidée : Retard de réplication
Un décalage Master/Slave a été suspecté, mais le fait de pouvoir rejoindre une partie après 10 minutes infirme cette piste.

### ✅ Hypothèse retenue : Race Condition (Ordre d'exécution)
Le bug provient de l'ordre séquentiel des événements lors du chevauchement de sockets :

1. **Connexion n°2 :** Écrit sur la base Master/Cache ➝ `Joueur X est en jeu (Partie A)`.
2. **Destruction WebSocket n°1 :** Pour laisser la place, le serveur tue l'ancienne socket.
3. **Nettoyage (onDisconnect) :** La fonction de déconnexion de la socket n°1 se déclenche. Elle a pour instruction de passer le statut à `pas en partie`.
4. **Écrasement de l'état :** Cet événement de déconnexion (issu de la socket 1) arrive *après* l'écriture de la connexion (socket 2).

> **État logique en base :** `Pas en partie` (écrit par la fermeture de la socket 1)
> **État réel du joueur :** `En jeu` (via la socket 2 active)

Le système de cache distribué conserve cette valeur erronée indéfiniment car aucun nouvel événement ne vient la corriger.

## ⚠️ Impact et Limites

**Impact :** Possibilité de jouer plusieurs parties simultanément (avantages déloyaux, farming, etc.).

**Limites connues :**
*   **Partie commencée :** Chevaucher les sockets *après* le début de la partie ne semble pas déclencher le bug (état stocké différemment ?).
*   **Contournement avancé :** Il est possible d'exploiter la faille sur une partie commencée en utilisant `GoToNextGame` (non patché) : chevaucher la socket de la nouvelle game, puis revenir à la partie initiale.

## 💡 Piste de correction

Modifier la logique de nettoyage lors d'une déconnexion forcée par chevauchement.

**Logique suggérée :**
Passer le statut du joueur en `pas en partie` **AVANT** d'accepter et d'écrire l'état de la nouvelle connexion (WebSocket n°2) sur cette même partie. S'assurer que l'événement de déconnexion de la socket 1 n'écrase pas l'état écrit par la socket 2.
