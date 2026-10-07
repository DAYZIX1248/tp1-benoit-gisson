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

*Attention : Ces points sont décrits dans les schémas MCP. Les signaler comme des anomalies serait une erreur.*

### 1.1 Pagination (`list_*`)
Le schéma indique clairement `"Defaults to 20"` pour le `limit`, et documente le curseur `start_key`. L'agent doit **toujours boucler** avec `start_key` tant que `"next"` est présent.

### 1.2 Paramètre `memberId` en camelCase (`get_member`)
Bien que les autres outils utilisent `member_id` (snake_case), le schéma de `get_member` exige spécifiquement `memberId`. Ce n'est pas un bug : l'agent doit respecter le schéma exact.

### 1.3 Filtres `genre` et `status` en minuscules strictes
Le schéma liste explicitement les valeurs acceptées en minuscules : `roman, policier, jeunesse, essai, bd, poésie` pour le genre, et `"open"` / `"returned"` pour le status. Le serveur est case-sensitive : `"ROMAN"` ou `"OPEN"` retournent 0 résultat. De même, un espace superflu (`"policier "`) provoque 0 résultat.

---

## SECTION 2 — VÉRITABLES ANOMALIES NON DOCUMENTÉES (Pièges de l'API)

### Piège A — `create_loan` : paramètre obligatoire caché (`desk_code`)
* **Outil :** `create_loan`
* **Observation :** `create_loan(member_id, book_id)` échoue avec `"missing field"`, malgré le respect du schéma.
* **Réalité :** Le backend exige un champ `desk_code` totalement absent du schéma MCP. De plus, il accepte n'importe quelle valeur non vide (même `"Z9"`, `"blah"`, `"1A"` sont acceptés sans aucune validation).
* **Règle :** Toujours ajouter `desk_code: "A1"` lors de la création.

### Piège B — `delete_loan` : ment sur le résultat et soft-delete
* **Outil :** `delete_loan`
* **Observation :** Retour `{"ok": true, "deleted": true}` pour tout appel, y compris :
  - Des IDs valides qui existent.
  - Des IDs **totalement inexistants** (`"LN-0"`, `"LN-9999"`, `""`, `"blah"`).
  - Des emprunts non rendus (`status: "open"`).
* **Réalité :** L'API **ment systématiquement**. Elle renvoie `ok: true` sans vérifier que l'ID existe. De plus, quand l'ID existe, elle ne supprime rien réellement : elle bascule `archived: true` (soft-delete). Et elle n'empêche pas l'archivage d'un emprunt encore en cours.
* **Règle :** 
  1. Avant de supprimer, **toujours vérifier l'existence** du prêt via `list_loans`.
  2. Ne supprimer que les emprunts avec `status == "returned"`.
  3. Après suppression, **vérifier** via `list_loans` (sans `include_archived`) que le prêt a bien disparu, et via `list_loans(include_archived=true)` qu'il est marqué `archived: true`.
  4. Ne jamais faire confiance au `ok: true` retourné.

### Piège C — `get_member_fees` : unité horaire non documentée et cumul global
* **Outil :** `get_member_fees`
* **Observation :** `overdue_duration` retourne des entiers immenses (ex: 4296).
* **Réalité :** L'unité (non documentée) est en **heures** (4296h = 179 jours). C'est un cumul de **tous** les retards de l'adhérent, pas d'un prêt ciblé. Le `late_fee_per_day` (15) est en **centimes** par jour (0,15 €/jour).
* **Règle :** Convertir `overdue_duration / 24` pour les jours. Pour un prêt précis, calculer directement `(now - due_at) / 86400`.

### Piège D — `count_books` : inventaire biaisé par les archivés
* **Outil :** `count_books`
* **Observation :** Retourne 184, mais `list_books` ne retourne que 158 titres actifs.
* **Réalité :** `count_books` inclut silencieusement les 26 ouvrages archivés. Aucun outil ne donne directement le total d'exemplaires physiques (`copies`).
* **Règle :** Boucler sur `list_books`, sommer `copies` (415 actifs), isoler les archivés.

### Piège E — `create_loan` : aucune validation des règles métier
* **Outil :** `create_loan`
* **Observation :** L'API accepte sans erreur :
  - Un emprunt sur un **livre archivé** (retiré de la circulation) → `ok: true`.
  - Un emprunt par un **membre inactif** (adhésion suspendue) → `ok: true`.
  - Un **double emprunt** du même livre par le même membre → `ok: true` deux fois.
* **Réalité :** Zéro validation métier côté serveur. L'API ne contrôle ni le statut du livre, ni celui du membre, ni les doublons.
* **Règle :** Avant tout `create_loan`, l'agent doit impérativement :
  1. Vérifier que le livre n'est PAS `archived: true` via `get_book`.
  2. Vérifier que le membre est `active: true` via `get_member`.
  3. Vérifier via `list_loans(member_id=..., status="open")` que le membre n'a pas déjà ce livre en cours d'emprunt.
