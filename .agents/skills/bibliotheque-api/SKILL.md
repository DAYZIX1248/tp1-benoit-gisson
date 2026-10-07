---
name: bibliotheque-api
description: >-
  Guide opérationnel de l'API MCP "bibliotheque-municipale". Distingue les comportements documentés (à respecter) des anomalies réelles non documentées de l'API.
---

# Guide Opérationnel — API Serveur MCP Médiathèque Municipale

Ce skill dicte à l'agent comment interagir correctement avec la médiathèque. Il distingue les comportements normaux (bien que surprenants) explicitement prévus par le schéma, et les véritables anomalies non documentées de l'API.

---

## 1. Comportements NORMAUX ET DOCUMENTÉS (À ne pas confondre avec des bugs)

*Attention : Ne signalez pas ces points comme des erreurs du serveur. Ils sont décrits dans les schémas MCP.*

* **Pagination (Outils `list_*`) :**  
  Le schéma indique clairement `Defaults to 20` pour le paramètre `limit`, et documente le `start_key`. L'agent ne doit pas traiter cela comme un piège, mais **toujours implémenter une boucle de pagination** avec `start_key` tant que `"next"` est fourni, notamment pour scanner l'intégralité des emprunts.
* **Casse du paramètre `memberId` (Outil `get_member`) :**  
  Bien que les autres outils utilisent `member_id`, le schéma de `get_member` exige spécifiquement `memberId` (camelCase). Ce n'est pas un bug : l'agent doit simplement lire et respecter le schéma exact.

---

## 2. VÉRITABLES ANOMALIES NON DOCUMENTÉES (Pièges de l'API)

### Piège A : `create_loan` (Paramètre obligatoire caché hors schéma)
* **Outil concerné :** `create_loan`
* **Ce que l'on observe :** Appeler `create_loan(member_id="...", book_id="...")` échoue avec `{"ok": false, "error": "missing field"}`, bien que l'inputSchema MCP n'exige que ces deux propriétés.
* **Ce que fait réellement le serveur :** Le backend exige le code guichet `desk_code` (ex: `"A1"` ou `"B2"`), totalement absent du schéma MCP.
* **Règle à appliquer :** Toujours injecter manuellement `desk_code: "A1"` lors de la création d'un emprunt.

### Piège B : `delete_loan` (Faux delete et absence de garde-fous)
* **Outil concerné :** `delete_loan`
* **Ce que l'on observe :** L'outil renvoie `{"ok": true, "deleted": true}`, mais l'emprunt reste visible si l'on ajoute `include_archived=true`. De plus, il accepte de "supprimer" un emprunt non rendu (`status: "open"`).
* **Ce que fait réellement le serveur :** Il n'effectue aucun hard-delete. Il réalise un **soft-delete** (`archived: true`) et ne vérifie aucune règle métier de restitution.
* **Règle à appliquer :** Filtrer strictement pour n'effacer que les emprunts ayant `status == "returned"`. Vérifier la disparition via `list_loans(member_id="...")` (doit être absent) et `list_loans(include_archived=true)` (doit être `archived: true`).

### Piège C : `get_member_fees` (Unité horaire et cumul global)
* **Outil concerné :** `get_member_fees`
* **Ce que l'on observe :** Le champ `overdue_duration` retourne des valeurs immenses (ex: 4296) non typées.
* **Ce que fait réellement le serveur :** Ce champ, non explicité dans le schéma, est exprimé en **heures** (4296h = 179 jours). Par ailleurs, il retourne la somme globale de **tous** les retards de l'adhérent, et non la durée d'un prêt ciblé.
* **Règle à appliquer :** Convertir les heures en jours (/ 24). Ne jamais attribuer ce chiffre global à un emprunt précis si l'adhérent a plusieurs prêts en retard. Pour un prêt précis, calculer le retard directement avec son timestamp `due_at`.

### Piège D : Différence d'inventaire (`count_books` vs `list_books`)
* **Outil concerné :** `count_books` et `list_books`
* **Ce que l'on observe :** `count_books` retourne 184 (sans paramètres). `list_books` retourne 158 titres actifs et ne fait aucune somme. 
* **Ce que fait réellement le serveur :** `count_books` indique la taille brute du catalogue incluant les retirés (`archived: true`). Le piège réside dans le concept "d'ouvrage" : aucun outil ne donne directement le volume d'exemplaires physiques (qui se lit dans `copies`).
* **Règle à appliquer :** Pour un inventaire exact, boucler sur `list_books`. Sommer le champ `copies` pour avoir le nombre d'exemplaires physiques (415 actifs), et faire la distinction claire avec les 26 ouvrages archivés.
