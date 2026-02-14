# 🐺 Wolfy Tools: Batch Generator & Multi-Account Manager

Ce projet est une suite d'outils d'automatisation pour la plateforme **Wolfy.net**. Il se compose de deux parties :
1.  **Backend (Node.js)** : Un serveur local capable de générer des comptes en masse, de valider les emails automatiquement via l'API Gmail et de configurer les comptes via des proxys.
2.  **Frontend (Userscript)** : Une interface UI injectée dans le jeu pour gérer, connecter et faire parler plusieurs comptes simultanément (supporte le Socket Polling via Proxy).

---

## 🏗️ Architecture et Fonctionnement

### Comment ça marche ?

1.  **Génération de comptes (`/batchRegister`)** :
    *   Le Frontend envoie une demande au Backend local.
    *   Le Backend lance un navigateur invisible (**Playwright**) et injecte votre Token Discord pour contourner l'écran de connexion manuelle.
    *   Il initie l'OAuth Wolfy, intercepte le code d'autorisation, et l'envoie à l'API Wolfy via un **Proxy Résidentiel** (pour éviter le ban IP).
    *   Il change l'email du compte et écoute l'API **Gmail** pour intercepter le lien de validation.
    *   Une fois validé, il renvoie le cookie de connexion (Token) au Frontend.

2.  **Gestion Multi-Comptes (Frontend)** :
    *   Le script utilise une implémentation personnalisée du protocole **Socket.IO** (Polling).
    *   Chaque requête réseau d'un "bot" passe par un **Worker Cloudflare** (CORS Proxy) pour masquer l'origine et permettre la multi-connexion depuis un seul navigateur.

---

## ⚙️ Partie 1 : Backend (Node.js)

### Prérequis
*   Node.js (v16+)
*   Un fournisseur de Proxys (ex: BrightData, IPRoyal..) format `user:pass@host:port`.
*   Des identifiants **Google Cloud Console** pour l'API Gmail (Client ID, Secret, Refresh Token).

### Installation
1.  Créez un dossier `wolfy-backend`.
2.  Ouvrez un terminal dans ce dossier et lancez :
    ```bash
    npm init -y
    npm install express playwright axios https-proxy-agent
    ```
3.  Créez un fichier `index.js` et collez le code ci-dessous.

### 📄 Code Source Backend (`index.js`)
* remplacer : votre-email@gmail.com, http://user:password@brd.superproxy.io:22225, VOTRE_CLIENT_ID_GOOGLE.apps.googleusercontent.com, VOTRE_CLIENT_SECRET_GOOGLE, VOTRE_REFRESH_TOKEN_GOOGLE

```javascript
// index.js
// Usage: node index.js

const express = require("express");
const { chromium } = require("playwright");
const axios = require("axios");
const { HttpsProxyAgent } = require("https-proxy-agent");
const { URL } = require("url");

const app = express();
const PORT = 3000;
app.use(express.json());

// --- CONFIGURATION A REMPLIR ---

// 1. PROXY (Obligatoire pour éviter les bans à l'inscription)
// Format : "http://user:pass@host:port"
const SINGLE_PROXY_RAW = "http://user:password@brd.superproxy.io:22225"; 

// 2. CONFIGURATION GMAIL (Pour la validation automatique des emails)
// Nécessite un projet Google Cloud avec Gmail API activé.
const GMAIL_CONFIG = {
    clientId: "VOTRE_CLIENT_ID_GOOGLE.apps.googleusercontent.com",
    clientSecret: "VOTRE_CLIENT_SECRET_GOOGLE",
    refreshToken: "VOTRE_REFRESH_TOKEN_GOOGLE"
};

// ------------------------------------

// CORS pour autoriser le script Tampermonkey
app.use((req, res, next) => {
    res.setHeader("Access-Control-Allow-Origin", "*");
    res.setHeader("Access-Control-Allow-Methods", "GET, POST, OPTIONS");
    res.setHeader("Access-Control-Allow-Headers", "Content-Type");
    next();
});

// --- Gestion Proxy ---
function normalizeProxyLine(line) {
    if (!line) return null;
    // (Fonction de normalisation simplifiée pour la doc)
    return line.trim(); 
}
const SINGLE_PROXY_CANONICAL = normalizeProxyLine(SINGLE_PROXY_RAW);

// --- Helpers HTTP avec Proxy ---
async function sendWithProxyWithRetries(proxyUrl, method, url, data = null, opts = {}, maxAttempts = 3) {
    if (!proxyUrl) return method === "post" ? axios.post(url, data, opts) : axios.get(url, opts);
    
    let lastErr = null;
    for (let attempt = 1; attempt <= maxAttempts; attempt++) {
        try {
            const agent = new HttpsProxyAgent(proxyUrl);
            const axiosOpts = { ...opts, httpsAgent: agent, timeout: 20000 };
            return method === "post" ? await axios.post(url, data, axiosOpts) : await axios.get(url, axiosOpts);
        } catch (e) {
            lastErr = e;
            await new Promise(r => setTimeout(r, 1000));
        }
    }
    throw lastErr;
}

// --- Helpers Gmail ---
async function getGmailAccessToken() {
    try {
        const resp = await axios.post("https://oauth2.googleapis.com/token", new URLSearchParams({
            client_id: GMAIL_CONFIG.clientId,
            client_secret: GMAIL_CONFIG.clientSecret,
            refresh_token: GMAIL_CONFIG.refreshToken,
            grant_type: "refresh_token",
        }).toString());
        if (resp.data?.access_token) return { ok: true, access_token: resp.data.access_token };
        return { ok: false, error: "no_access_token" };
    } catch (e) {
        return { ok: false, error: e.message };
    }
}

function base64UrlDecodeToUtf8(str) {
    return Buffer.from(str.replace(/-/g, "+").replace(/_/g, "/"), "base64").toString("utf8");
}

async function getConfirmUrlFromGmail(alias, timeoutMs = 120000) {
    const tokenResp = await getGmailAccessToken();
    if (!tokenResp.ok) return { ok: false, reason: "gmail_token_failed" };
    
    const accessToken = tokenResp.access_token;
    const q = encodeURIComponent("to:" + alias);
    const start = Date.now();
    
    console.log(`[GMAIL] Waiting email for ${alias}...`);
    
    while (Date.now() - start < timeoutMs) {
        try {
            const listRes = await axios.get(`https://gmail.googleapis.com/gmail/v1/users/me/messages?q=${q}&maxResults=1`, {
                headers: { Authorization: `Bearer ${accessToken}` }
            });
            
            if (listRes.data.messages?.length) {
                const msgId = listRes.data.messages[0].id;
                const getRes = await axios.get(`https://gmail.googleapis.com/gmail/v1/users/me/messages/${msgId}?format=full`, {
                    headers: { Authorization: `Bearer ${accessToken}` }
                });
                
                const snippet = getRes.data.snippet || "";
                // Recherche lien de confirmation Wolfy
                const match = snippet.match(/https?:\/\/wolfy\.net[^\s"'<>]*confirm[^\s"'<>]*/i);
                if (match) return { ok: true, confirmUrl: match[0] };
            }
        } catch (e) { console.log("Gmail poll err", e.message); }
        await new Promise(r => setTimeout(r, 5000));
    }
    return { ok: false, reason: "timeout" };
}

// --- Helpers Wolfy ---
async function postRegisterViaProxy(proxyUrl, accessToken, username) {
    try {
        const payload = { accessToken, username, lang: "fr" };
        const resp = await sendWithProxyWithRetries(proxyUrl, "post", "https://wolfy.net/api/auth/discord/register", payload, {
            headers: { "User-Agent": "Mozilla/5.0", "Content-Type": "application/json" }
        });
        return { ok: true, status: resp.status, headers: resp.headers };
    } catch (e) { return { ok: false, error: e.message }; }
}

async function performWolfyFollowups(proxyUrl, wolfyCookie, newEmailAlias) {
    const results = {};
    const agent = proxyUrl ? new HttpsProxyAgent(proxyUrl) : null;
    const config = { headers: { Cookie: `wolfy=${wolfyCookie}`, Origin: "https://wolfy.net" }, httpsAgent: agent };

    // 1. Unlink Discord
    try { await axios.post("https://wolfy.net/api/auth/unlink/discord", {}, config); results.unlink = true; } catch(e) { results.unlink = false; }
    
    // 2. Change Password
    try { await axios.post("https://wolfy.net/api/settings/password", { oldPass: "", newPass: "saltay123" }, config); results.pwd = true; } catch(e) { results.pwd = false; }

    // 3. Change Email & Validate
    if (newEmailAlias) {
        try {
            await axios.post("https://wolfy.net/api/settings/email", { email: newEmailAlias }, config);
            const val = await getConfirmUrlFromGmail(newEmailAlias);
            if (val.ok) {
                // Confirm URL via proxy
                let path = new URL(val.confirmUrl).pathname.replace("/confirm", "/api/auth/confirm");
                await sendWithProxyWithRetries(proxyUrl, "post", "https://wolfy.net" + path, {}, { headers: { "User-Agent": "Mozilla/5.0" }});
                results.email = "validated";
            }
        } catch(e) { results.email = "error"; }
    }
    return results;
}

// --- Route Principale ---
app.post("/batchRegister", async (req, res) => {
    const { usernames = [], discordToken, emailBase = "votre-email@gmail.com" } = req.body;
    if (!discordToken || !usernames.length) return res.status(400).json({ error: "Missing inputs" });

    const browser = await chromium.launch({ headless: true });
    const results = [];

    try {
        for (const username of usernames) {
            console.log(`Processing ${username}...`);
            const context = await browser.newContext();
            const page = await context.newPage();

            // Interception pour ne pas charger la page finale
            await page.route("**/*", (route) => {
                const url = route.request().url();
                if (url.includes("/auth/finalize") || url.includes("/auth/confirm")) route.abort();
                else route.continue();
            });

            // Injection Token Discord
            await page.addInitScript((token) => {
                window.localStorage.setItem("token", `"${token}"`);
            }, discordToken);

            // Navigation OAuth
            try {
                // Note : URL OAuth officielle de Wolfy
                await page.goto("https://discord.com/oauth2/authorize?client_id=501842028569559061&response_type=code&scope=email+identify&redirect_uri=https%3A%2F%2Fwolfy.net%2Fauth%2Ffinalize", { timeout: 10000, waitUntil: "domcontentloaded" });
                
                // Click "Authorize" si nécessaire
                try { await page.click('button:has-text("Authorize")', { timeout: 2000 }); } catch {}
                
                await page.waitForTimeout(1500); // Attente redirection
            } catch (e) {}

            // Extraction CODE
            const finalUrl = page.url();
            const code = finalUrl.match(/[?&]code=([^&]+)/)?.[1];

            if (code) {
                // Inscription via Proxy
                const reg = await postRegisterViaProxy(SINGLE_PROXY_CANONICAL, code, username);
                if (reg.ok) {
                    const cookie = reg.headers['set-cookie']?.[0]?.match(/wolfy=([^;]+)/)?.[1];
                    if (cookie) {
                        const alias = emailBase.replace("@", `+${Date.now()}@`);
                        await performWolfyFollowups(SINGLE_PROXY_CANONICAL, cookie, alias);
                        results.push({ username, wolfyToken: decodeURIComponent(cookie) });
                    }
                }
            }
            await context.close();
        }
    } catch (e) { console.error(e); } finally { await browser.close(); }

    res.json({ ok: true, results });
});

app.listen(PORT, () => console.log(`Backend running on port ${PORT}`));
```

---

## 🖥️ Partie 2 : Frontend (Userscript Tampermonkey)

### Prérequis
*   Extension **Tampermonkey** installée sur votre navigateur.
*   Un **Worker Cloudflare** déployé pour servir de proxy CORS (Le script attend une URL type `https://worker.dev/?url=`). Sans cela, la connexion aux sockets échouera.

### 📄 Code Source Userscript
* remplacer : https://votre-worker-cloudflare.workers.dev/?url=, ["Salut", "Ca va ?", "Test", "Fin"] (par des mots offensants qui trigger le filtre de wolfy), VOTRE_TOKEN_DISCORD_ICI
```javascript
// ==UserScript==
// @name         Wolfy Tools UI
// @namespace    http://tampermonkey.net/
// @version      2.3
// @description  Interface de gestion multi-comptes et générateur batch pour Wolfy.
// @match        https://wolfy.net/*
// @run-at       document-idle
// @grant        none
// ==/UserScript==

(function () {
  'use strict';

  // ---------------- CONFIGURATION ----------------

  // URL de votre Worker Cloudflare (Proxy CORS)
  // Le code du worker doit faire un fetch(decodeURIComponent(params.get('url')))
  const proxyBase = "https://votre-worker-cloudflare.workers.dev/?url=";

  // Token Discord (Utilisateur) utilisé pour la génération de comptes
  const discordToken = "VOTRE_TOKEN_DISCORD_ICI";

  // URL du backend Node.js (doit tourner en local)
  const backendUrl = "http://localhost:3000/batchRegister";

  // Messages envoyés par la fonction "Marteau"
  const HAMMER_MESSAGES = ["Salut", "Ca va ?", "Test", "Fin"];

  // ---------------- ÉTAT ----------------
  let tokenInfos = [];
  const tokenSessions = []; // SIDs socket
  const tokenProxyUrls = [];
  const keepAliveIntervals = [];
  let gameId = null;
  let instanceId = null;

  // ---------------- HELPERS RÉSEAU ----------------
  const fetchProxy = (url, options = {}) => fetch(proxyBase + encodeURIComponent(url), options);

  // Parsing des frames Socket.IO (Protocole 42)
  function extract42Frames(text) {
    const frames = [];
    let pos = 0;
    while (true) {
      const idx = text.indexOf('42[', pos);
      if (idx === -1) break;
      const open = text.indexOf('[', idx + 2);
      if (open === -1) { pos = idx + 2; continue; }
      // Logique simplifiée extraction JSON...
      let end = text.indexOf('}]', open); // approximation pour la doc
      if (end === -1) end = text.length;
      frames.push(text.slice(open, end + 2)); // Ajustement nécessaire selon structure réelle
      pos = end + 1;
    }
    return frames;
  }

  // Boucle de polling (Simule un client Socket.IO en HTTP Polling)
  async function startPolling(i, proxyUrl, token) {
    try {
      const res = await fetch(proxyUrl, { headers: { 'X-Cookie': `wolfy=${token}` } });
      const text = await res.text();
      // On parse le SID si présent pour maintenir la session
      const sidMatch = text.match(/\{["']sid["']\s*:\s*["']([^"']+)["']\}/);
      if (sidMatch && !tokenSessions[i]) tokenSessions[i] = sidMatch[1];
      
      // On relance immédiatement pour avoir du temps réel
      if (tokenSessions[i]) setTimeout(() => startPolling(i, proxyUrl, token), 0);
    } catch (e) {
      setTimeout(() => startPolling(i, proxyUrl, token), 2000);
    }
  }

  // ---------------- FONCTIONS ACTIONS ----------------
  
  async function connectToken(i) {
    const token = tokenInfos[i]?.token;
    if (!gameId || !instanceId) return alert("Entrez dans une partie d'abord !");
    
    // 1. Initialisation (Handshake)
    const baseUrl = `https://wolfy.net/instance/${instanceId}/socket.io/`;
    let url = `${baseUrl}?gameId=${gameId}&EIO=4&transport=polling&t=${Date.now()}`;
    
    // Fetch via Proxy pour obtenir le SID
    const resText = await fetch(proxyBase + encodeURIComponent(url), { 
        headers: { 'X-Cookie': `wolfy=${token}` } 
    }).then(r => r.text());
    
    const sid = JSON.parse(resText.substring(resText.indexOf('{'))).sid;
    tokenSessions[i] = sid;
    
    const sessionUrl = proxyBase + encodeURIComponent(url + `&sid=${sid}`);
    tokenProxyUrls[i] = sessionUrl;

    // 2. Connexion (Engine.IO upgrade & Subscribe)
    await fetch(sessionUrl, { method: 'POST', headers: { 'X-Cookie': `wolfy=${token}` }, body: '40' });
    await fetch(sessionUrl, { method: 'POST', headers: { 'X-Cookie': `wolfy=${token}` }, body: '42' + JSON.stringify(["subscribe", { gameId }]) });

    // 3. Keep Alive (Ping toutes les 20s)
    keepAliveIntervals[i] = setInterval(() => {
        fetch(sessionUrl, { method: 'POST', headers: { 'X-Cookie': `wolfy=${token}` }, body: '3' }).catch(()=>{});
    }, 20000);

    // 4. Lecture
    startPolling(i, sessionUrl, token);
  }

  async function sendMessage(i, txt) {
    if(!tokenSessions[i]) return;
    const body = '42' + JSON.stringify(["chat", { text: txt, private: false }]);
    fetch(tokenProxyUrls[i], { method: 'POST', headers: { 'X-Cookie': `wolfy=${tokenInfos[i].token}` }, body });
  }

  // ---------------- UI BUILDER ----------------
  function createUI() {
    if (document.getElementById("wolfy-tools-ui")) return;
    
    const div = document.createElement("div");
    div.id = "wolfy-tools-ui";
    div.style.cssText = "position:fixed; top:50px; left:10px; background:#111; color:#fff; padding:10px; border-radius:8px; z-index:99999; max-height:80vh; overflow:auto; border:1px solid #444;";
    
    // Header
    div.innerHTML = `<h3>🐺 Wolfy Tools <small style="font-size:10px; cursor:pointer" onclick="this.parentElement.parentElement.remove()">❌</small></h3>`;
    
    // Batch Generator Zone
    const batchZone = document.createElement("div");
    batchZone.innerHTML = `
        <div style="margin-bottom:10px; padding-bottom:10px; border-bottom:1px solid #333;">
            <input type="text" id="wt-pseudos" placeholder="Pseudos (séparés par ,)" style="width:100%; box-sizing:border-box; margin-bottom:5px; background:#222; color:#fff; border:1px solid #444; padding:5px;">
            <button id="wt-gen-btn" style="width:100%; background:#4CAF50; border:none; color:white; padding:5px; cursor:pointer;">Générer Comptes</button>
        </div>
        <div id="wt-list" style="display:flex; flex-direction:column; gap:5px;"></div>
    `;
    div.appendChild(batchZone);
    document.body.appendChild(div);

    // Event Listener Génération
    document.getElementById("wt-gen-btn").onclick = async () => {
        const raw = document.getElementById("wt-pseudos").value;
        if(!raw) return;
        const btn = document.getElementById("wt-gen-btn");
        btn.disabled = true; btn.innerText = "Génération en cours...";
        
        try {
            const res = await fetch(backendUrl, {
                method: "POST",
                headers: {"Content-Type":"application/json"},
                body: JSON.stringify({ usernames: raw.split(","), discordToken })
            });
            const data = await res.json();
            if(data.ok) {
                data.results.forEach(r => addTokenToList(r.wolfyToken, r.username));
            } else {
                alert("Erreur Backend: " + JSON.stringify(data));
            }
        } catch(e) { alert("Erreur: Vérifiez que node index.js tourne bien."); }
        btn.disabled = false; btn.innerText = "Générer Comptes";
    };
  }

  function addTokenToList(token, name) {
    const idx = tokenInfos.length;
    tokenInfos.push({ token, name });
    
    const row = document.createElement("div");
    row.style.cssText = "display:flex; justify-content:space-between; align-items:center; background:#222; padding:5px; border-radius:4px;";
    row.innerHTML = `<span style="font-size:12px; width:80px; overflow:hidden;">${name}</span>`;
    
    const actions = document.createElement("div");
    
    const btnJoin = document.createElement("button"); btnJoin.innerText = "➕";
    btnJoin.onclick = () => connectToken(idx);
    
    const btnChat = document.createElement("button"); btnChat.innerText = "📩";
    btnChat.onclick = () => { const m = prompt("Message:"); if(m) sendMessage(idx, m); };
    
    const btnHammer = document.createElement("button"); btnHammer.innerText = "🔨";
    btnHammer.onclick = async () => { for(let msg of HAMMER_MESSAGES) { sendMessage(idx, msg); await new Promise(r=>setTimeout(r,100)); } };

    actions.append(btnJoin, btnChat, btnHammer);
    row.appendChild(actions);
    document.getElementById("wt-list").appendChild(row);
  }

  // Injection auto si on est en jeu
  function checkUrl() {
    if(location.pathname.startsWith("/game/") && !gameId) {
        // Récupération ID partie via API
        const code = location.pathname.split("/")[2];
        fetchProxy(`https://wolfy.net/api/game/${code}`).then(r=>r.json()).then(d => {
            gameId = d.id; instanceId = d.instanceId;
            createUI();
        });
    }
  }
  
  setInterval(checkUrl, 2000);

})();
```
