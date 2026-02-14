## 📄 Résumé

Ce script UserScript a pour objectif d'identifier et d'afficher le pseudonyme des joueurs utilisant le mode anonyme (rang "Alpha") ainsi que celui des modérateurs sur le jeu Wolfy.

Il contourne la fonctionnalité d'anonymat en se basant sur une base de données externe d'IDs de joueurs et en l'associant aux avatars affichés dans le jeu.

## ⚙️ Principe de Fonctionnement

Le script opère en trois étapes distinctes et efficaces :

#### 1. Collecte de Données (Phase d'Initialisation)
Au chargement de la page, le script utilise `GM_xmlhttpRequest` pour contacter une base de données hébergée sur **GitHub Gist**, scrapée lorsque l'endpoint api/autocomplete/query était vulnérable et sans rate limit (sécurisé maintenant).
*   **Source :** Un fichier `players_filtered.ndjson` contenant une liste d'objets JSON, un par ligne.
*   **Filtrage :** Le script parcourt ce fichier et ne conserve que les joueurs ayant le rang "Alpha" ou "Modérateur".
*   **Stockage :** Il charge ces informations (ID, pseudo, rang) dans une `Map` en mémoire pour un accès quasi-instantané.

#### 2. Détection des Joueurs (Phase d'Observation)
Le script utilise un `MutationObserver` pour surveiller en temps réel toute modification du DOM (l'affichage de la page).
*   **Cible :** Il recherche spécifiquement les balises `<img>` dont l'URL source correspond au format des avatars du jeu (ex: `.../api/skin/render/user.svg?id=PLAYER_ID`).
*   **Extraction :** Il extrait l'identifiant unique (`PLAYER_ID`) de l'URL de chaque avatar affiché à l'écran.
*   **Vérification :** Il vérifie si cet ID est présent dans la `Map` chargée à l'étape 1.

#### 3. Affichage de l'Information (Phase de Révélation)
Si un ID correspond à une entrée dans la base de données, le script injecte dynamiquement un élément HTML au-dessus de l'avatar.
*   **Contenu :** Cet élément affiche le vrai pseudonyme du joueur.
*   **Style :** Le style de l'étiquette est différentié :
    *   **Rouge** pour les Modérateurs.
    *   **Bleu/Gris** pour les joueurs Alpha.
*   **Nettoyage :** Le script inclut également un "nettoyeur" qui supprime l'étiquette si le joueur quitte la partie (et donc que son avatar est retiré de la page), afin d'éviter les éléments fantômes.

## 💣 Analyse de l'Impact et des Implications

*   **Contournement Total de l'Anonymat :** Ce script rend la fonctionnalité "Alpha" complètement inefficace. Tout utilisateur du script peut identifier immédiatement les joueurs anonymes ciblés par la base de données.
*   **Dépendance à une Source Externe :** L'efficacité du script repose entièrement sur la mise à jour et la disponibilité de la base de données sur GitHub Gist. Si le fichier est supprimé, obsolète ou si l'URL change, le script devient inutile.
*   **Ciblage Potentiel :** Permet à des joueurs de cibler (ou d'éviter) spécifiquement des joueurs connus (streamers, modérateurs, etc.) même s'ils tentent de se cacher.

## 💡 Pistes de Contre-Mesure

Pour neutraliser ce type de script, plusieurs solutions peuvent être envisagées côté serveur :

1.  **Changer les URLs des Avatars :** La solution la plus simple. Modifier la structure de l'URL des avatars (ex: ne plus y inclure l'ID de manière directe) casserait instantanément la méthode de détection du script.
2.  **Utiliser des IDs de Session :** Remplacer l'ID utilisateur permanent par un identifiant temporaire, unique à chaque partie, dans les URLs publiques comme celles des avatars.

---

### 🔧 Code Source Complet

<details>
<summary>Cliquez pour voir le code du UserScript</summary>

```javascript
// ==UserScript==
// @name         Wolfy Alpha Detector
// @namespace    jsp
// @version      1.1
// @description  Affiche le pseudo des joueurs Alpha dans les parties Wolfy
// @author       jsp
// @match        https://wolfy.net/*
// @grant        GM_xmlhttpRequest
// @connect      gist.githubusercontent.com
// ==/UserScript==


(function () {
    'use strict';

    const ALPHA_RANK = "Alpha";
    const MODO_RANK = "Modérateur";
    const DATA_URL = "https://gist.githubusercontent.com/Salt4yy/4b9ad31dec3bb4b27917936083ae2115/raw/a8cfa74be69da66050a083f148341abf420cd718/players_filtered.ndjson";

    let idToUsernameMap = new Map();


function displayAlphaTagOutsideAvatar(username, imgElement, rank) {
    const parent = imgElement.parentElement;
    if (!parent) return null;
    const isSvg = imgElement.src.includes(".svg");

    const box = document.createElement("div");
    box.textContent = username;
    Object.assign(box.style, {
        position: "absolute",
        background: rank === MODO_RANK ? "#a00" : "#222",
        color: rank === MODO_RANK ? "#fff" : "#aabbee",
        padding: "2px 6px",
        borderRadius: "4px",
        fontSize: "10px",
        fontWeight: "bold",
        fontFamily: "Arial, sans-serif",
        pointerEvents: "none",
        zIndex: 9999,
        whiteSpace: "nowrap",
        boxShadow: "0 0 5px rgba(0,0,0,0.7)",
        top: isSvg ? "-40px" : "-20px",
        left: "50%",
        transform: "translateX(-50%)"
    });

    if (getComputedStyle(parent).position === "static") {
        parent.style.position = "relative";
    }

    parent.appendChild(box);
    return box;
}

    function loadNDJSON(callback) {
        GM_xmlhttpRequest({
            method: "GET",
            url: DATA_URL,
            onload: function (response) {
                const lines = response.responseText.split("\n");
                for (const line of lines) {
                    if (line.trim().length === 0) continue;
                    try {
                        const obj = JSON.parse(line);
                        if (obj.rank === ALPHA_RANK || obj.rank === MODO_RANK) {
    idToUsernameMap.set(obj.id, {
        username: obj.username,
        rank: obj.rank
    });
}
                    } catch (e) {
                        console.warn("Erreur parsing NDJSON :", e);
                    }
                }
                console.log("Alpha map loaded:", idToUsernameMap);
                callback();
            }
        });
    }

function cleanupOnImageRemoval() {
    const observer = new MutationObserver(mutations => {
        for (const mutation of mutations) {
            for (const node of mutation.removedNodes) {
                if (node.nodeType === 1 && node.tagName === "IMG") {
                    const img = node;
                    if (img.src.includes("/api/skin/render/user.svg?id=")) {
                        const parent = img.parentElement;
                        if (parent) {
                            const box = parent.querySelector("div[data-alpha-box='true']");
                            if (box) box.remove();
                        }
                    }
                }
            }
        }
    });

    observer.observe(document.body, {
        childList: true,
        subtree: true
    });
}


function observeImages() {
    const observer = new MutationObserver((mutations) => {
        mutations.forEach(mutation => {

            mutation.removedNodes.forEach(node => {
                if (node.nodeType === Node.ELEMENT_NODE) {
                    const imgs = node.querySelectorAll?.("img[src*='/api/skin/render/user.svg?id=']");
                    imgs?.forEach(img => {
                        if (img.saltayAlphaBox) {
                            img.saltayAlphaBox.remove();
                            delete img.saltayAlphaBox;
                        }
                    });


                    if (node.matches?.("img[src*='/api/skin/render/user.svg?id=']") && node.saltayAlphaBox) {
                        node.saltayAlphaBox.remove();
                        delete node.saltayAlphaBox;
                    }
                }
            });


            const imgs = document.querySelectorAll("img[src*='/api/skin/render/user.png?id='], img[src*='/api/skin/render/user.svg?id=']");
            imgs.forEach(img => {
                try {
                    const url = new URL(img.src);
                    const id = url.searchParams.get("id");
                    if (idToUsernameMap.has(id) && !img.dataset.saltayTagged) {
                        const { username, rank } = idToUsernameMap.get(id);
const box = displayAlphaTagOutsideAvatar(username, img, rank);
                        img.saltayAlphaBox = box;
                        img.dataset.saltayTagged = "true";
                    }
                } catch (e) {
                    console.warn("Erreur lecture ID de l’image:", e);
                }
            });
        });
    });

    observer.observe(document.body, {
        childList: true,
        subtree: true
    });
}

    loadNDJSON(() => {
    observeImages();
    cleanupOnImageRemoval();
    console.log("Script actif.");
});
})();
