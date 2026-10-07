---
name: bibliotheque-api
description: >-
  Guide opérationnel de l'API MCP "bibliotheque-municipale". Distingue les comportements documentés (à respecter) des anomalies réelles non documentées de l'API.
  À activer impérativement avant toute manipulation du catalogue, des adhérents ou des emprunts.
---

# Guide Opérationnel — API Serveur MCP Médiathèque Municipale

Ce skill dicte à l'agent comment interagir correctement avec la médiathèque. Il distingue les comportements normaux explicitement prévus par le schéma, et les véritables anomalies non documentées de l'API.

---

## SECTION 1 — Comportements DOCUMENTÉS (Ne pas signaler comme des bugs)

*Attention : Ces points sont décrits dans les schémas MCP. Les signaler comme des anomalies serait une erreur selon la règle "Tout signaler ne paie pas".*

### 1.1 Pagination standard (`list_*`)
Le schéma indique clairement `"Defaults to 20"` pour le `limit`, et documente le curseur `start_key`. L'agent doit **toujours boucler** avec `start_key` tant que `"next"` est présent et que la liste d'items n'est pas vide.

### 1.2 Paramètre `memberId` en camelCase (`get_member`)
Bien que les autres outils utilisent `member_id` (snake_case), le schéma de `get_member` exige spécifiquement `memberId`. Ce n'est pas un bug : l'agent doit respecter le schéma exact.

### 1.3 Filtres `genre` et `status` en minuscules strictes
Le schéma liste explicitement les valeurs acceptées en minuscules : `roman, policier, jeunesse, essai, bd, poésie` pour le genre, et `"open"` / `"returned"` pour le status. Le serveur est case-sensitive : `"ROMAN"` ou `"OPEN"` retournent 0 résultat.

---

## SECTION 2 — VÉRITABLES ANOMALIES NON DOCUMENTÉES (Pièges de l'API)

### Piège A — `create_loan` : paramètre obligatoire caché (`desk_code`)
* **Outil :** `create_loan`
* **Observation :** `create_loan(member_id, book_id)` échoue avec `"missing field"`, malgré le respect strict de l'inputSchema.
* **Réalité :** Le backend exige un champ `desk_code` totalement absent du schéma MCP. De plus, il accepte n'importe quelle valeur non vide (`"Z9"`, `"blah"`, `"1A"`…) sans aucune validation du code guichet.
* **Règle :** Toujours ajouter `desk_code: "A1"` lors de la création d'un emprunt.

### Piège B — `delete_loan` : ment systématiquement et soft-delete
* **Outil :** `delete_loan`
* **Observation :** Retour `{"ok": true, "deleted": true}` pour **tout appel**, y compris :
  - Des IDs totalement inexistants (`"LN-0"`, `"LN-9999"`, `""`, `"blah"`).
  - Des emprunts non rendus (`status: "open"`).
* **Réalité :** L'API **ment systématiquement**. Elle renvoie `ok: true` sans vérifier que l'ID existe. Quand l'ID existe, elle ne supprime rien : elle bascule `archived: true` (soft-delete).
* **Règle :** Toujours vérifier l'existence préalable du prêt via `list_loans`. Ne supprimer que les emprunts avec `status == "returned"`. Vérifier la disparition via `list_loans` (sans `include_archived`) et `list_loans(include_archived=true)`. Ne jamais se fier au `ok: true`.

### Piège C — `get_member_fees` : unité horaire non documentée et cumul global
* **Outil :** `get_member_fees`
* **Observation :** `overdue_duration` retourne des entiers démesurés (ex: 4296).
* **Réalité :** L'unité (non documentée) est en **heures** (4296h = 179 jours). C'est un cumul de **tous** les retards de l'adhérent, pas d'un prêt ciblé. Le `late_fee_per_day` (15) est en **centimes** par jour (0,15 €/jour).
* **Règle :** Convertir `overdue_duration / 24` pour les jours. Pour un prêt précis, calculer directement `(now - due_at) / 86400`.

### Piège D — `count_books` : inventaire trompeur incluant les archivés
* **Outil :** `count_books`
* **Observation :** Retourne 184, mais `list_books` ne retourne que 158 titres actifs. De plus, passer des arguments comme `include_archived: false` ou `genre: roman` est totalement ignoré.
* **Réalité :** `count_books` inclut silencieusement les 26 ouvrages archivés. Aucun outil ne donne directement le total d'exemplaires physiques (`copies`).
* **Règle :** Boucler sur `list_books`, sommer `copies` (415 actifs), et isoler les 26 archivés.

### Piège E — `create_loan` : absence totale de contrôle métier
* **Outil :** `create_loan`
* **Observation :** L'API accepte silencieusement (`ok: true`) :
  - Un emprunt sur un **livre archivé** (retiré de la circulation).
  - Un emprunt par un **membre inactif** (adhésion suspendue).
  - Un **double emprunt** du même livre par le même membre.
* **Réalité :** Zéro validation métier côté serveur.
* **Règle :** Avant tout `create_loan`, vérifier que le livre n'est pas `archived: true`, que le membre est `active: true`, et qu'il n'a pas déjà ce livre en cours d'emprunt.

### Piège F — `search_books` : renvoie des livres archivés et sensible aux accents
* **Outil :** `search_books`
* **Observation :** 
  - `search_books("Été")` retourne 20 résultats, tandis que `search_books("Ete")` retourne **0 résultat** (aucun repliement d'accents / accent folding).
  - La recherche retourne silencieusement des **ouvrages archivés** (`archived: true`, par exemple `BK-1002`) sans avertissement ni paramètre pour les exclure.
* **Réalité :** Contrairement à `list_books` qui masque les archivés par défaut, `search_books` sonde l'intégralité de la base sans filtre d'archivage possible.
* **Règle :** Utiliser l'orthographe exacte avec accents, et filtrer obligatoirement côté client les résultats pour écarter les livres dont `archived === true`.

### Piège G — `list_loans` : token `next` trompeur sur résultat vide
* **Outil :** `list_loans`
* **Observation :** Appeler `list_loans` pour un adhérent sans prêt ou inexistant (`MB-9999`) retourne `items: []` avec néanmoins un token `next: "MjA="`.
* **Réalité :** Le générateur de pagination renvoie systématiquement un curseur `next` même lorsqu'il n'y a plus aucune donnée.
* **Règle :** Toute boucle de pagination doit s'interrompre si `next` est absent **OU si `len(items) == 0`** pour éviter une boucle infinie de requêtes.
