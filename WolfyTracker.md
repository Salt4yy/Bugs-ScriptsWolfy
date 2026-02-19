# 📊 Wolfy Leaderboard Tracker

Ce script surveille en temps réel le classement de Wolfy pour détecter les changements d'XP. Lorsqu'un joueur gagne de l'XP, cela signifie qu'il est en train de jouer, et une notification est immédiatement envoyée via un Webhook Discord.

### 🛠️ Configuration
Dans le code, modifie les variables suivantes :
- `WEBHOOK_URL` : Ton lien de webhook Discord.
- `cookie` : Ton token Wolfy (à récupérer dans l'inspecteur d'élément, onglet Application -> Cookies).

### 📦 Installation
Il est nécessaire d'installer la bibliothèque `axios` pour que le script fonctionne :
```bash
npm install axios
```

### 🚀 Lancement
```bash
node leaderboard_tracker.js
```

---

Le code :
```javascript
const axios = require('axios');


const WEBHOOK_URL = "URL DU WEBHOOK ICI"; 
const API_URL = "https://wolfy.net/api/leaderboard";
const INTERVAL = 15000; 


const HEADERS = {
    "accept": "application/json, text/plain, */*",
    "accept-encoding": "gzip, deflate, br, zstd",
    "accept-language": "fr-FR,fr;q=0.9",
    "cache-control": "no-cache",
    
    "cookie": "NEXT_LOCALE=fr; wolfy=TOKEN WOLFY ICI,
    "pragma": "no-cache",
    "priority": "u=1, i",
    "referer": "https://wolfy.net/fr/leaderboard/",
    "sec-ch-ua": "\"Not/A)Brand\";v=\"8\", \"Chromium\";v=\"143\", \"Edge\";v=\"143\"",
    "sec-ch-ua-mobile": "?0",
    "sec-ch-ua-platform": "\"Windows\"",
    "sec-fetch-dest": "empty",
    "sec-fetch-mode": "cors",
    "sec-fetch-site": "same-origin",
    "user-agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36 Edg/143.0.0.0"
};


let previousData = new Map();
let isFirstRun = true;


async function sendDiscordNotification(username, oldXp, newXp) {
    const diff = newXp - oldXp;
    const embed = {
        title: "UN MODO JOUE",
        color: 0x9b59b6, 
        fields: [
            { name: "Joueur", value: username, inline: true },
        ],
        timestamp: new Date()
    };

    try {
       
        await axios.post(WEBHOOK_URL, { 
            content: "@everyone", 
            embeds: [embed] 
        });
        console.log(`[DISCORD] Notification envoyée pour ${username}`);
    } catch (err) {
        console.error("[ERREUR] Impossible d'envoyer le webhook:", err.message);
    }
}


async function checkLeaderboard() {
    try {
        console.log(`[${new Date().toLocaleTimeString()}] Vérification du leaderboard...`);
        
        const response = await axios.get(API_URL, { headers: HEADERS });
        const currentPlayers = response.data;

        
        if (isFirstRun) {
            currentPlayers.forEach(player => {
                previousData.set(player.id, player.xp);
            });
            isFirstRun = false;
            console.log("Données initiales chargées.");
            return;
        }

        
        for (const player of currentPlayers) {
            const oldXp = previousData.get(player.id);

            
            if (oldXp !== undefined && oldXp !== player.xp) {
                console.log(`Changement pour ${player.username}: ${oldXp} -> ${player.xp}`);
                await sendDiscordNotification(player.username, oldXp, player.xp);
                
               
                previousData.set(player.id, player.xp);
            }
            
            else if (oldXp === undefined) {
                previousData.set(player.id, player.xp);
            }
        }

    } catch (error) {
        console.error("[ERREUR API]", error.message);
        
        if (error.response && (error.response.status === 401 || error.response.status === 403)) {
            console.error("cookies invalides ou expirés.");
        }
    }
}
