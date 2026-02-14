# 🐺 Wolfy Research & Tools

Ce dépôt regroupe un ensemble de **Preuves de Concept (PoC)**, de **scripts d'automatisation (UserScripts)** et de **rapports de vulnérabilités** concernant la plateforme de jeu **Wolfy.net**.

Il documente plusieurs mois de recherche sur l'architecture WebSocket, la logique serveur et les faiblesses de l'API du jeu.

> ⚠️ **DISCLAIMER / AVERTISSEMENT**
>
> Ce projet est publié à des fins purement **éducatives** et de **recherche en sécurité**.
> L'utilisation de ces scripts pour tricher, nuire à l'expérience des autres joueurs (DoS client) ou attaquer les serveurs est strictement interdite par les Conditions d'Utilisation de Wolfy.
> L'auteur ne saurait être tenu responsable des bannissements de comptes ou des dommages causés par l'utilisation de ces outils.

---

## 📂 Contenu du Dépôt

### 🛠️ Outils & Scripts (UserScripts)

Ces outils améliorent l'interface ou automatisent des actions via Tampermonkey.

| Fichier | Description | Type |
| :--- | :--- | :--- |
| **[`GenerateurCompteWolfy.md`](./GenerateurCompteWolfy.md)** | Système complet (Node.js + UI) pour générer des comptes en masse, valider les emails et gérer une flotte de bots via Proxys. | **Botting** |
| **[`AnonymeDetector.md`](./AnonymeDetector.md)** | Script permettant de révéler le vrai pseudo des joueurs en mode "Anonyme" (Alpha) et des Modérateurs via une base de données externe. | **OSINT** |
| **[`PlaceSpam.md`](./PlaceSpam.md)** | Interface UI moderne et flottante pour spammer les changements de place dans un salon. | **Utility** |

### 🐛 Rapports de Vulnérabilités & Exploits

Analyses techniques de failles logiques et critiques découvertes sur l'infrastructure.

| Fichier | Sévérité | Description |
| :--- | :--- | :--- |
| **[`WebsocketDesynchro.md`](./WebsocketDesynchro.md)** | 🔴 **Critique** | **Race Condition** permettant de contourner la sécurité de session unique et de jouer plusieurs parties simultanément sur le même compte. |
| **[`PatchedBugs.md`](./PatchedBugs.md)** | 🔴 **Critique** | Documentation de crashs serveurs (DoS), crashs clients via vote invalide et corruption de la composition des parties. |
| **[`NextGameBug.md`](./NextGameBug.md)** | 🟠 **Moyenne** | Faille logique permettant de forcer la redirection "GoToNextGame" alors qu'une partie est encore en cours. |

---

## 🚀 Installation & Utilisation

### Prérequis
1.  Un navigateur récent (Chrome, Firefox, Brave).
2.  L'extension **[Tampermonkey](https://www.tampermonkey.net/)**.
3.  Pour le *Générateur de Comptes* uniquement : **[Node.js](https://nodejs.org/)** installé sur votre machine.

### Comment utiliser un script ?
1.  Cliquez sur le fichier `.md` de l'outil qui vous intéresse dans la liste ci-dessus.
2.  Faites défiler jusqu'à la section **"Code Source Complet"**.
3.  Copiez le bloc de code JavaScript.
4.  Ouvrez Tampermonkey, créez un **Nouveau Script**, collez le code et sauvegardez (`Ctrl+S`).

---

## 👤 Auteur

**Salt4yy** - Recherche de vulnérabilités, Reverse Engineering WebSocket & Développement d'outils.
