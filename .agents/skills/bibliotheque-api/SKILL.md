---
name: bibliotheque-api
description: >-
  Guide opérationnel exhaustif de l'API MCP "bibliotheque-municipale".
  Recense les 11 pièges réels et non documentés de l'API ainsi que les comportements normaux à respecter.
---

# Guide Opérationnel — API Serveur MCP Médiathèque Municipale

Ce skill dicte à l'agent comment interagir avec la médiathèque sans faillir. Il recense les **11 véritables pièges non documentés** de l'API et rappelle les comportements normaux prévus par le schéma.

---

## SECTION 1 — Comportements DOCUMENTÉS (À ne pas signaler comme des bugs)

*Règle d'or : "Tout signaler ne paie pas". Ces comportements sont écrits dans les schémas MCP.*
1. **Pagination standard (`list_*`) :** `limit` vaut 20 par défaut et `start_key` est requis pour paginer. C'est documenté.
2. **Casse de `memberId` (`get_member`) :** Bien que les autres outils utilisent `member_id`, le schéma exige `memberId` en camelCase. C'est documenté.
3. **Casse stricte des statuts et genres :** Le schéma liste explicitement les valeurs en minuscules (`roman`, `open`...). Passer `"ROMAN"` ou `"OPEN"` renvoie 0 résultat, ce qui est conforme au schéma.

---

## SECTION 2 — LES 11 VÉRITABLES PIÈGES NON DOCUMENTÉS DE L'API

### Piège 1 — `create_loan` : Paramètre obligatoire caché (`desk_code`)
* **Outil concerné :** `create_loan`
* **Ce que l'on observe :** Appel avec `member_id` et `book_id` renvoie `{"ok": false, "error": "missing field"}` bien que seuls ces deux champs soient déclarés dans le schéma MCP.
* **Comportement réel :** Le backend exige le code guichet `desk_code` (absent du schéma). Il accepte n'importe quelle chaîne non vide (`"A1"`, `"B2"`, `"Z9"`) sans contrôle de validité.
* **Règle :** Toujours fournir `desk_code: "A1"`.

### Piège 2 — `delete_loan` : Mensonge systématique sur les IDs inexistants
* **Outil concerné :** `delete_loan`
* **Ce que l'on observe :** Appeler `delete_loan` avec des identifiants totalement fictifs (`"LN-0"`, `"LN-9999"`, `"blah"`) renvoie `{"ok": true, "deleted": true}`.
* **Comportement réel :** L'API ment systématiquement : elle valide la suppression sans vérifier si le prêt existe en base.
* **Règle :** Ne jamais croire le retour de `delete_loan`. Toujours vérifier au préalable l'existence du prêt via `list_loans`.

### Piège 3 — `delete_loan` : Faux delete (Soft-delete masqué)
* **Outil concerné :** `delete_loan`
* **Ce que l'on observe :** Après un appel `delete_loan` réussi, l'emprunt n'apparaît plus dans `list_loans` normal, mais réapparaît dès qu'on passe `include_archived=true`.
* **Comportement réel :** L'outil ne supprime aucune ligne : il bascule le statut sur `archived: true`.
* **Règle :** Pour prouver la suppression, vérifier que le prêt a disparu du registre actif et qu'il porte la mention `archived: true` dans `list_loans(include_archived=true)`.

### Piège 4 — `delete_loan` : Effacement frauduleux des dettes financières
* **Outil concerné :** `delete_loan` & `get_member_fees`
* **Ce que l'on observe :** Si l'on "supprime" un prêt non rendu en retard (`status: "open"`), la dette de l'adhérent dans `get_member_fees` tombe immédiatement à 0 € (`balance_due: 0`).
* **Comportement réel :** L'API permet d'archiver un prêt en cours sans restitution du livre, et le module financier ignore les prêts archivés, effaçant ainsi frauduleusement la dette.
* **Règle :** Interdiction d'appeler `delete_loan` sur un prêt dont `status != "returned"`.

### Piège 5 — `get_member_fees` : Unités horaires et cumul global non documentés
* **Outil concerné :** `get_member_fees`
* **Ce que l'on observe :** `overdue_duration` retourne des valeurs démesurées (ex: 4296).
* **Comportement réel :** L'unité non documentée est en **heures** (4296h = 179 jours). Ce chiffre additionne en bloc les retards de **tous** les emprunts de l'adhérent. Le taux journalier (15) est en **centimes** (0,15 €/j).
* **Règle :** Diviser `overdue_duration` par 24 pour obtenir les jours. Pour un prêt individuel, calculer la durée directement avec son timestamp `due_at`.

### Piège 6 — `count_books` : Inventaire trompeur et insensibilité aux filtres
* **Outil concerné :** `count_books`
* **Ce que l'on observe :** Retourne obstinément 184, y compris lorsqu'on lui passe `include_archived: false` ou un genre précis.
* **Comportement réel :** Compte silencieusement les 26 ouvrages archivés, ne compte pas les exemplaires physiques (`copies`), et ignore tous les paramètres de filtrage.
* **Règle :** Boucler sur `list_books`, sommer le champ `copies` pour les exemplaires (415 actifs), et isoler les 26 archivés.

### Piège 7 — `create_loan` : Absence totale de contrôle du statut (Archive & Inactif)
* **Outil concerné :** `create_loan`
* **Ce que l'on observe :** `create_loan` réussit avec `ok: true` lors d'un emprunt sur un **livre archivé** ou par un **membre inactif** (`active: false`).
* **Comportement réel :** Aucune validation d'intégrité métier n'est exécutée côté serveur.
* **Règle :** Vérifier avant l'emprunt que `book.archived === false` et que `member.active === true`.

### Piège 8 — `create_loan` : Dépassement illimité du stock physique (Overbooking)
* **Outil concerné :** `create_loan`
* **Ce que l'on observe :** Sur un livre ayant `copies: 2`, l'API accepte d'enregistrer 5 emprunts simultanés sans aucune erreur.
* **Comportement réel :** L'API ne décrémente pas le nombre de copies et n'empêche pas l'emprunt au-delà du stock physique disponible.
* **Règle :** Comparer le nombre d'emprunts ouverts sur un livre au champ `copies` de `get_book` avant de valider un prêt.

### Piège 9 — `search_books` : Inclusion masquée des archives et sensibilité stricte aux accents
* **Outil concerné :** `search_books`
* **Ce que l'on observe :** `search_books("Été")` retourne 20 résultats mais `search_books("Ete")` retourne 0 résultat (aucun accent-folding). De plus, la recherche retourne silencieusement des livres archivés (`BK-1002`).
* **Comportement réel :** Contrairement à `list_books`, `search_books` sonde toute la base sans filtrer les archives et applique un filtre textuel strict sans normalisation d'accents.
* **Règle :** Chercher avec l'orthographe accentuée exacte, et filtrer les résultats côté client avec `archived === false`.

### Piège 10 — `list_loans` : Curseur `next` trompeur sur collection vide
* **Outil concerné :** `list_loans`
* **Ce que l'on observe :** Interroger un adhérent sans aucun prêt (`MB-9999`) retourne `items: []` avec néanmoins un token `next: "MjA="`.
* **Comportement réel :** Le générateur de pagination renvoie un jeton `next` même quand la page est vide.
* **Règle :** Arrêter toute pagination si `next` est absent **OU si `len(items) == 0`** pour éviter les boucles infinies.

### Piège 11 — Tous outils : Incohérence temporelle tripartite (3 formats incompatibles)
* **Outils concernés :** `get_member`, `get_book`, `list_loans`, `get_member_fees`
* **Ce que l'on observe :** L'API mélange 3 formats temporels radicalement incompatibles :
  - Adhérent (`joined_at`) : Chaîne française `DD/MM/YYYY` (ex: `"05/12/2024"`).
  - Livre (`added_at`) : Chaîne ISO 8601 UTC (`"2023-04-17T09:00:00.000Z"`).
  - Emprunt (`started_at`, `due_at`) : Entier UNIX timestamp en secondes (`1777798800`).
  - Retard (`overdue_duration`) : Entier en heures.
* **Comportement réel :** Absence totale de normalisation temporelle dans le socle de l'API.
* **Règle :** Toujours convertir explicitement les dates vers des timestamps UNIX ou des objets Date natifs avant toute comparaison.
