# Wolfy Place Spam (Interface Utilisateur Parfaite)

Ce script utilisateur (UserScript) ajoute une interface flottante et moderne sur le site `wolfy.net` pour automatiser l'envoi de messages de changement de place (spam) dans le jeu. L'interface permet d'activer ou de désactiver le spam et d'en régler la vitesse.

## ⚙️ Fonctionnement général

Le script fonctionne en deux parties principales :
1.  **Interface Utilisateur (UI)** : Un panneau de contrôle flottant, déplaçable, qui permet de gérer le spam.
2.  **Logique (Core & Manager)** : Un moteur qui intercepte la communication du jeu (via WebSocket) pour pouvoir envoyer des messages de changement de place automatisés.

### Interactions

*   **Déplacement** : L'interface peut être déplacée par un simple glisser-déposer. La position est sauvegardée dans le `localStorage` du navigateur pour qu'elle réapparaisse au même endroit lors des prochaines visites.
*   **Activation** : Un clic sur le bouton active ou désactive le spam.
*   **Réglage de la vitesse** : Le curseur permet d'ajuster l'intervalle de temps entre chaque message de changement de place envoyé.

### 🔧 Code Source Complet

<details>
<summary>Cliquez pour voir le code du UserScript</summary>

```javascript
// ==UserScript==
// @name         Wolfy Place spam (Perfect UI)
// @namespace    http://tampermonkey.net/
// @version      7.1
// @description  Tools for Wolfy with modern centered UI
// @author       moi
// @match        https://wolfy.net/*
// @grant        none
// @run-at       document-start
// ==/UserScript==

(function() {
    'use strict';

    let core = null;
    const UI_ID = 'wolfy-overlay-ui';
    const KEY_POS = 'wolfy_ui_pos';

    function initUI() {
        if (!document.body) {
            window.addEventListener('DOMContentLoaded', initUI);
            return;
        }

        const existing = document.getElementById(UI_ID + '-container');
        if (existing) existing.remove();

        const style = document.createElement('style');
        style.innerHTML = `
            #${UI_ID}-container {
                position: fixed;
                z-index: 999999;
                background: rgba(15, 15, 15, 0.9);
                backdrop-filter: blur(12px);
                padding: 18px;
                border-radius: 14px;
                border: 1px solid rgba(231, 76, 60, 0.4);
                box-shadow: 0 10px 40px rgba(0,0,0,0.5);
                display: flex;
                flex-direction: column;
                gap: 14px;
                user-select: none;
                min-width: 190px;
                transition: border-color 0.3s;
                font-family: 'Segoe UI', Roboto, sans-serif;
            }
            #${UI_ID}-container.on {
                border-color: rgba(46, 204, 113, 0.6);
            }
            #${UI_ID} {
                color: #e74c3c;
                font-weight: 900;
                font-size: 13px;
                letter-spacing: 1.5px;
                cursor: pointer;
                text-align: center;
                text-transform: uppercase;
            }
            #${UI_ID}.on {
                color: #2ecc71;
            }

            .wolfy-slider-container {
                display: flex;
                flex-direction: column;
                gap: 10px;
            }
            .wolfy-speed-label {
                color: #aaa;
                font-size: 10px;
                font-weight: 700;
                display: flex;
                justify-content: space-between;
                letter-spacing: 0.5px;
            }
            .wolfy-speed-value {
                color: #fff;
            }

            /* --- STYLE DU SLIDER --- */
            .wolfy-slider {
                -webkit-appearance: none;
                width: 100%;
                height: 4px; /* Épaisseur de la barre */
                background: rgba(255, 255, 255, 0.1);
                border-radius: 2px;
                outline: none;
                margin: 10px 0; /* Espace pour ne pas couper la boule */
            }

            /* La boule (Thumb) - Webkit */
            .wolfy-slider::-webkit-slider-thumb {
                -webkit-appearance: none;
                appearance: none;
                width: 16px;
                height: 16px;
                background: #e74c3c;
                border-radius: 50%;
                cursor: pointer;
                /* CALCUL POUR CENTRAGE : (HauteurBarre / 2) - (HauteurBoule / 2) */
                /* (4 / 2) - (16 / 2) = 2 - 8 = -6px */
                margin-top: -6px;
                transition: transform 0.1s ease, background 0.3s;
                box-shadow: 0 0 8px rgba(231, 76, 60, 0.4);
            }

            #${UI_ID}-container.on .wolfy-slider::-webkit-slider-thumb {
                background: #2ecc71;
                box-shadow: 0 0 12px rgba(46, 204, 113, 0.5);
            }

            .wolfy-slider::-webkit-slider-thumb:hover {
                transform: scale(1.15);
            }
        `;
        document.head.appendChild(style);

        const container = document.createElement('div');
        container.id = UI_ID + '-container';

        const btn = document.createElement('div');
        btn.id = UI_ID;
        btn.textContent = 'Spam Status: OFF';

        const sliderCont = document.createElement('div');
        sliderCont.className = 'wolfy-slider-container';

        const labelRow = document.createElement('div');
        labelRow.className = 'wolfy-speed-label';
        labelRow.innerHTML = `<span>VITESSE</span><span class="wolfy-speed-value" id="ws-val">1ms</span>`;

        const slider = document.createElement('input');
        slider.type = 'range';
        slider.className = 'wolfy-slider';
        slider.min = '1';
        slider.max = '1000';
        slider.value = '1';

        slider.addEventListener('input', (e) => {
            const val = e.target.value;
            document.getElementById('ws-val').textContent = `${val}ms`;
            if (core && core.manager) core.manager.speed = parseInt(val);
        });

        slider.addEventListener('mousedown', (e) => e.stopPropagation());

        sliderCont.appendChild(labelRow);
        sliderCont.appendChild(slider);
        container.appendChild(btn);
        container.appendChild(sliderCont);

        // Position initiale avec padding
        const saved = localStorage.getItem(KEY_POS);
        if (saved) {
            const p = JSON.parse(saved);
            container.style.top = p.top;
            container.style.left = p.left;
        } else {
            container.style.bottom = '40px';
            container.style.left = '40px';
        }

        document.body.appendChild(container);

        // Drag & Drop
        let drag = false, moved = false, x, y, ix, iy;
        container.addEventListener('mousedown', (e) => {
            if (e.button !== 0) return;
            drag = true; moved = false;
            x = e.clientX; y = e.clientY;
            const r = container.getBoundingClientRect();
            ix = r.left; iy = r.top;
            container.style.bottom = 'auto'; container.style.left = ix + 'px'; container.style.top = iy + 'px';
        });
        window.addEventListener('mousemove', (e) => {
            if (!drag) return;
            const dx = e.clientX - x, dy = e.clientY - y;
            if (Math.abs(dx) > 2 || Math.abs(dy) > 2) moved = true;
            container.style.left = (ix + dx) + 'px';
            container.style.top = (iy + dy) + 'px';
        });
        window.addEventListener('mouseup', () => {
            if (!drag) return;
            drag = false;
            if (moved) {
                const r = container.getBoundingClientRect();
                localStorage.setItem(KEY_POS, JSON.stringify({top: r.top + 'px', left: r.left + 'px'}));
            } else {
                if (core && core.manager) core.manager.toggle();
            }
        });
    }

    function renderState(state) {
        const btn = document.getElementById(UI_ID);
        const container = document.getElementById(UI_ID + '-container');
        if (!btn || !container) return;
        if (state) {
            btn.classList.add('on');
            container.classList.add('on');
            btn.textContent = 'Spam Status: ON';
        } else {
            btn.classList.remove('on');
            container.classList.remove('on');
            btn.textContent = 'Spam Status: OFF';
        }
    }

    class Core {
        constructor() {
            this.socket = undefined;
            this.manager = new Manager(this);
            this.hookWS();
        }
        hookWS() {
            const _WS = window.WebSocket;
            const self = this;
            window.WebSocket = function(...args) {
                const ws = new _WS(...args);
                self.hook(ws);
                return ws;
            };
            window.WebSocket.prototype = _WS.prototype;
            const _send = window.WebSocket.prototype.send;
            window.WebSocket.prototype.send = new Proxy(_send, {
                apply: (tgt, thisArg, args) => {
                    if (self && !self.socket) self.hook(thisArg);
                    return tgt.apply(thisArg, args);
                }
            });
        }
        hook(ws) {
            if (this.socket === ws) return;
            this.socket = ws;
            ws.addEventListener('message', (e) => this.read(e));
            ws.addEventListener('close', () => { this.socket = undefined; });
        }
        read(msg) {
            let d = msg.data;
            if (typeof d !== 'string' || !d.includes('42[')) return;
            try {
                d = JSON.parse(d.slice(2));
                if (d[0] === 'settings' && d[1]?.places) this.manager.update(d[1].places);
                else if (d[0] === 'updatePlaces') this.manager.update(d[1]);
            } catch (err) {}
        }
    }

    class Manager {
        constructor(core) {
            this.core = core;
            this.active = false;
            this.speed = 1;
            this.free = Array.from({length: 30}, (_, i) => i);
            this.loop();
        }
        toggle() { this.active = !this.active; renderState(this.active); }
        update(data) {
            this.free = [];
            for (let i = 0; i < data.length; i++) if (!data[i]) this.free.push(i);
        }
        emit(p) {
            if (this.core.socket?.readyState === 1) {
                this.core.socket.send("42" + JSON.stringify(["updatePlace", { "place": p }]));
            }
        }
        loop() {
            const run = () => {
                if (this.active && this.core.socket) {
                    const target = this.free.length ? this.free[Math.floor(Math.random() * this.free.length)] : Math.floor(Math.random() * 20);
                    this.emit(target);
                }
                setTimeout(run, this.speed);
            };
            run();
        }
    }

    initUI();
    core = new Core();
})();
