## 📄 Description du Bug

Un joueur peut envoyer manuellement l'événement WebSocket `goToNextGame` à n'importe quel moment, même si sa partie actuelle est toujours en cours.

Le serveur traite cette requête sans vérifier si la partie est terminée, ce qui téléporte le joueur vers un nouveau salon de jeu tout en le maintenant dans sa partie initiale.

## 🛠️ Étapes de Reproduction

1.  Rejoindre et lancer une partie A.
2.  Pendant que la partie A est en cours (après le lancement), envoyer manuellement le message WebSocket suivant :
    ```
    42["goToNextGame",{}]
    ```
3.  Le client est immédiatement redirigé vers un nouveau salon de jeu (Partie B).

## 🆚 Comportement Actuel vs Attendu

| Type | Description |
| :--- | :--- |
| **Actuel** | Le serveur accepte la requête `goToNextGame` à tout moment. Le joueur est alors présent et actif simultanément dans la Partie A (qui se poursuit) et dans le salon de la Partie B. |
| **Attendu** | Le serveur ne devrait traiter l'événement `goToNextGame` qu'à la fin d'une partie (sur l'écran des résultats). Si la partie est en cours, la requête devrait être ignorée ou rejetée. |

## 💣 Analyse de l'Impact

Cette vulnérabilité a deux conséquences principales :

#### 1. Jeu simultané sur plusieurs parties (Impact Élevé)
Le joueur peut être **actif dans deux parties en même temps**. Il peut lancer la partie B tout en jouant la partie A. Palie aux faiblesses du bug de Desynchro qui ne marche plus si fait sur une partie déjà lancée.

#### 2. Recréation rapide de partie ("Remake") (Impact Moyen)
Les joueurs peuvent utiliser cette commande pour quitter un salon et en recréer un instantanément avec la même composition, sans avoir à annuler la partie manuellement.
*   **Avant le lancement :** Permet de "remake" la partie plus vite.
*   **Après le lancement :** Permet de quitter une partie en cours pour en relancer une autre immédiatement, abandonnant les autres joueurs.

---

### 💡 Piste de Correction

Côté serveur, ajouter une condition pour valider l'état de la partie avant d'exécuter la logique de `goToNextGame`. L'événement ne devrait être autorisé que si l'état de la partie du joueur est "Terminée".

### 🔧 Outil de Reproduction (Script)

Ce bug a été identifié et peut être facilement reproduit à l'aide du script Tampermonkey ci-dessous, qui ajoute un bouton "Remake" à l'interface.

<details>
<summary>Cliquez pour voir le code du UserScript</summary>

```javascript
// ==UserScript==
// @name         Wolfy - Remake Button
// @namespace    http://tampermonkey.net/
// @version      1.3
// @description  Bouton Remake
// @author       moi hihi
// @match        https://wolfy.net/*
// @grant        none
// @run-at       document-start
// ==/UserScript==

(function() {
    'use strict';

    let socket = null;
    const btnId = 'wx-rmk-btn';

    const style = document.createElement('style');
    style.innerHTML = `
        #${btnId} {
            position: fixed;
            top: 20px;
            left: 50%;
            transform: translateX(-50%);
            z-index: 999999;
            min-width: 160px;
            padding: 12px 30px;
            box-sizing: border-box;
            background: #1a1a2e;
            border: 2px solid #5e53d5;
            border-radius: 50px;
            box-shadow: 0 4px 12px rgba(0,0,0,0.4);
            color: #fff;
            font-family: 'Montserrat', sans-serif;
            font-size: 14px;
            font-weight: 800;
            text-align: center;
            letter-spacing: 1px;
            text-transform: uppercase;
            white-space: nowrap;
            line-height: 1;
            cursor: not-allowed;
            opacity: 0.6;
            transition: all 0.2s cubic-bezier(0.25, 0.8, 0.25, 1);
            user-select: none;
        }
        #${btnId}.ready {
            cursor: pointer;
            opacity: 1;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            border-color: transparent;
        }
        #${btnId}.ready:hover {
            transform: translateX(-50%) scale(1.05);
            box-shadow: 0 6px 20px rgba(118, 75, 162, 0.5);
        }
        #${btnId}.sent {
            background: #27ae60 !important;
            border-color: transparent !important;
            box-shadow: 0 0 15px rgba(39, 174, 96, 0.4);
        }
    `;
    document.head.appendChild(style);

    const NativeWebSocket = window.WebSocket;
    window.WebSocket = function(...args) {
        const ws = new NativeWebSocket(...args);
        if (args && args.includes("socket.io")) {
            socket = ws;
            ws.addEventListener('open', () => toggleState(true));
            ws.addEventListener('close', () => toggleState(false));
        }
        return ws;
    };

    function sendRemake() {
        if (!socket || socket.readyState !== 1) return;
        socket.send('42["goToNextGame",{}]');
        const btn = document.getElementById(btnId);
        const originalText = btn.innerText;
        btn.innerText = "C'EST PARTI !";
        btn.classList.add('sent');
        setTimeout(() => {
            btn.innerText = originalText;
            btn.classList.remove('sent');
        }, 1200);
    }

    function toggleState(active) {
        const btn = document.getElementById(btnId);
        if (!btn) return;
        active ? btn.classList.add('ready') : btn.classList.remove('ready');
    }

    function init() {
        if (document.getElementById(btnId)) return;
        const btn = document.createElement('button');
        btn.id = btnId;
        btn.innerText = "↻ REMAKE";
        btn.onclick = sendRemake;
        document.body.appendChild(btn);
    }

    if (document.readyState === "loading") {
        document.addEventListener("DOMContentLoaded", init);
    } else {
        init();
    }
})();
