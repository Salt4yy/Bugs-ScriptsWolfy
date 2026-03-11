### **Faille : Fuite d'ID réel via l'événement Socket `game_create`**

*   **Composant :** Socket du Hub Wolfy (`/hub`).
*   **Vulnérabilité :** Incohérence dans l'application du système d'anonymisation des IDs.
*   **Description :** Bien que les IDs soient désormais chiffrés/masqués dans la liste globale des parties (fetch initial) pour empêcher le traquage, l'événement de diffusion en temps réel `game_create` (envoyé via Socket.io lors de la création d'une nouvelle partie) contient l'ID original de l'administrateur dans les champs `adminId` et `admin.id`.
*   **Payload incriminé :**
    ```json
    ["game_create", {
        "id": "...",
        "adminId": "ID_REEL_NON_MASQUE",
        "admin": { "id": "ID_REEL_NON_MASQUE", ... }
    }]
    ```
*   **Impact :** Contournement total du système d'anonymat. Un script peut monitorer le flux de la socket en continu pour lier l'ID réel d'un joueur à ses parties créées (disponible dans AnonymeDetector.md de ce même repo), permettant son traquage persistant malgré l'activation du mode anonyme.
*   **Note technique :** La faille est spécifique au message de *broadcast* ; une nouvelle connexion socket (rafraîchissement complet) renvoie des données correctement anonymisées.
