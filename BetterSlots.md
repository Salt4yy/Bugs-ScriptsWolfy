# Wolfy BetterSlots

## Le Besoin

Sur la plateforme Wolfy, l'interface de personnalisation des skins impose des restrictions ergonomiques basées sur l'état actuel du slot sélectionné. Lorsqu'un slot est déjà équipé (soit normalement, soit anonymement), le moteur de rendu React supprime les boutons d'action correspondants dans le DOM.

### Limitations identifiées :
1. **Masquage conditionnel** : Impossibilité d'équiper directement en mode anonyme un slot déjà équipé normalement sans passer par une phase de déséquipement.
2. **Accessibilité restreinte** : Les fonctions d'équipement (`equipSlot` et `equipSlotAnonymous`) sont encapsulées dans un store interne difficilement accessible via la console standard.
3. **Réactivité restrictive** : Le cycle de rendu de React nettoie dynamiquement la zone de validation, empêchant toute modification statique du DOM.

## La Solution proposée

Le script propose une couche d'abstraction logique qui s'intercale entre le moteur de données du jeu et l'interface utilisateur. Il force la mise à disposition des options d'équipement manquantes en fonction de l'état réel du slot, tout en respectant l'identité visuelle native de la plateforme.

### 📄 Code Source

```javascript
// ==UserScript==
// @name         Wolfy BetterSlots
// @namespace    http://tampermonkey.net/
// @version      2.0
// @description  Boutons pour équiper normalement et en slot ano en même temps
// @author       Saltay
// @match        https://wolfy.net/*
// @run-at       document-start
// @grant        none
// ==/UserScript==

(function() {
    'use strict';

    let skinStore = null;

    const originalDefineProperty = Object.defineProperty;
    Object.defineProperty = function(obj, prop, desc) {
        if (prop === 'slotSelected') skinStore = obj;
        return originalDefineProperty.apply(this, arguments);
    };

    const style = document.createElement('style');
    style.textContent = `
        div[class*="Skin_validationButtons"] {
            display: flex !important;
            visibility: visible !important;
            opacity: 1 !important;
        }
        div[class*="Skin_validationButtons"] > div:not([data-forced]):not([class*="Skin_validate"]):not([class*="Skin_cancel"]) {
            display: none !important;
        }
        .btn-anon-dark {
            background-color: #2c3e50 !important;
        }
    `;
    document.head.appendChild(style);

    function createBtn(text, isAnon) {
        const btn = document.createElement('div');
        btn.dataset.forced = "true";
        btn.className = 'Skin_equip__j4TG7' + (isAnon ? ' btn-anon-dark' : '');
        btn.style.cursor = 'pointer';
        btn.style.margin = '5px';
        btn.style.userSelect = 'none';

        if (isAnon) {
            btn.innerHTML = `<img src="/static/img/icons/anonymous.svg" class="Skin_icon___95tm" style="width:20px; margin-right:8px;"> <span>${text}</span>`;
            btn.onclick = () => skinStore.equipSlotAnonymous(skinStore.slotSelected);
        } else {
            btn.innerText = text;
            btn.onclick = () => skinStore.equipSlot(skinStore.slotSelected);
        }

        return btn;
    }

    function updateUI() {
        const container = document.querySelector('div[class*="Skin_validationButtons"]');
        if (!container || !skinStore?.slotSelected) return;

        const slot = skinStore.slotSelected;
        const isNormal = slot.equiped;
        const isAnon = slot.anonymous;

        const stateId = `${slot.id}-${isNormal}-${isAnon}`;
        if (container.dataset.state === stateId) return;
        container.dataset.state = stateId;

        container.querySelectorAll('[data-forced]').forEach(el => el.remove());

        if (isNormal && isAnon) {
            return;
        } else if (isNormal && !isAnon) {
            container.appendChild(createBtn("Équiper (anonyme)", true));
        } else if (!isNormal && isAnon) {
            container.appendChild(createBtn("Équiper", false));
        } else {
            container.appendChild(createBtn("Équiper", false));
            container.appendChild(createBtn("Équiper (anonyme)", true));
        }

        container.classList.add('Skin_active__aKK0D');
    }

    setInterval(updateUI, 200);

    window.addEventListener('mousedown', () => {
        if (!skinStore) {
            const el = document.querySelector('div[class*="Skin_pageContent"]');
            if (!el) return;

            const key = Object.keys(el).find(k => k.startsWith('__reactFiber$'));
            let fiber = el[key];
            while (fiber) {
                if (fiber.memoizedProps?.skin?.equipSlot) {
                    skinStore = fiber.memoizedProps.skin;
                    break;
                }
                fiber = fiber.return;
            }
        }
    });
})();
