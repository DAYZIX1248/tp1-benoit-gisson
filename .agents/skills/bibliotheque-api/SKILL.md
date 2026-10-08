---
name: bibliotheque-api
description: >-
  Guide opérationnel et de contournement des 11 pièges de l'API MCP "bibliotheque-municipale".
  À activer impérativement avant toute manipulation du catalogue, des adhérents ou des emprunts.
---

# Guide Opérationnel — API Serveur MCP Médiathèque Municipale

Ce skill répertorie les **11 pièges réels (P1 à P11)** du serveur MCP ainsi que les comportements normaux documentés afin d'éviter tout faux positif.

---

## 1. COMPORTEMENTS NORMAUX (Ne pas signaler comme des anomalies)

*Règle d'or : « Tout signaler ne paie pas ». Ces comportements sont conformes à la documentation des outils.*

1. **Archives exclues par défaut dans `list_books` :** La description indique formellement que les exemplaires archivés sont exclus sauf si `include_archived: true`.
2. **Casse de `memberId` dans `get_member` :** Bien que les autres outils utilisent `member_id`, le schéma de `get_member` spécifie explicitement `memberId` en camelCase.
3. **Filtre appliqué avant pagination :** Le serveur applique bien le filtre en base de données avant la pagination (la première page contient déjà des identifiants élevés). Les pages vides proviennent uniquement de la non-terminaison de `next` (P1).

---

## 2. LES 11 PIÈGES RÉELS DE L'API (P1 à P11)

### P1 — Curseur `next` jamais null après la fin de collection
* **Outil :** `list_books`, `list_loans`, `list_members`
* **Observation :** Après la dernière page de données, l'API continue indéfiniment de renvoyer `{"items": [], "next": "..."}`.
* **Comportement réel :** Le token `next` n'est jamais mis à `null` quand la collection est épuisée.
* **Règle :** Toujours arrêter la pagination dès que `len(items) < limit` ou `len(items) == 0`. Ne jamais boucler uniquement sur la présence de `next`.

### P2 — Paramètre `limit` plafonné silencieusement à 50
* **Outil :** `list_books`, `list_loans`, `list_members`
* **Observation :** Si l'on demande `limit: 100` ou `limit: 500`, l'API ne renvoie jamais plus de 50 éléments.
* **Comportement réel :** Le backend bride la taille maximale de page à 50 sans avertissement ni mention dans le schéma.
* **Règle :** Utiliser `limit: 50` d'emblée et toujours prévoir une boucle de pagination pour tout volume supérieur à 50.

### P3 — Écart d'inventaire : `count_books` (184) ≠ `list_books` (158)
* **Outil :** `count_books` & `list_books`
* **Observation :** `count_books` renvoie 184 alors que `list_books` (sans archives) n'en liste que 158.
* **Comportement réel :** `count_books` compte tous les titres enregistrés (actifs + 26 archivés), alors que `list_books` filtre les archivés. De plus, `count_books` ne compte que les titres uniques et ignore le total d'exemplaires physiques (`copies`).
* **Règle :** Pour un inventaire en circulation, boucler sur `list_books` sans archives (158 titres, 415 exemplaires physiques). N'annoncer 184 titres (490 exemplaires) qu'en précisant qu'il s'agit du patrimoine total incluant les 26 ouvrages retirés.

### P4 — `create_loan` exige le paramètre caché non documenté `desk_code`
* **Outil :** `create_loan`
* **Observation :** Appel avec `{"member_id": "...", "book_id": "..."}` échoue avec `{"ok": false, "error": "missing field"}` bien que conformes au schéma MCP.
* **Comportement réel :** Le backend exige le code guichet `desk_code` (ex: `"A1"`), totalement omis dans l'inputSchema.
* **Règle :** Toujours renseigner `desk_code: "A1"` lors de la création d'un emprunt.

### P5 — `delete_loan` effectue un soft-delete et ment systématiquement
* **Outil :** `delete_loan`
* **Observation :** Renvoie `{"ok": true, "deleted": true}` pour n'importe quel ID (même inexistant comme `"LN-0"`). Cependant, le prêt existe toujours avec `archived: true` dans `list_loans(include_archived=true)`.
* **Comportement réel :** L'outil ne supprime aucune ligne : il bascule le statut sur `archived: true` sans vérifier l'existence de l'ID ni la restitution du livre.
* **Règle :** Vérifier au préalable l'existence du prêt. Ne supprimer que les prêts avec `status == "returned"`. Prouver l'opération en montrant la disparition du registre actif et la présence de `archived: true` avec `include_archived=true`.

### P6 — `get_member_fees` : unités horaires, centimes et cumul global
* **Outil :** `get_member_fees`
* **Observation :** `overdue_duration` affiche des entiers comme 4296, et `balance_due` vaut 26.85.
* **Comportement réel :** L'unité (non documentée) de `overdue_duration` est en **heures** (4296 h = 179 jours). Ce chiffre cumule **tous** les retards de l'adhérent. Le taux `late_fee_per_day` (15) est en **centimes** (0,15 €/jour).
* **Règle :** Diviser `overdue_duration` par 24 pour obtenir les jours. Pour un prêt individuel, calculer `(date_ref - due_at) / 86400`.

### P7 — Horloge interne du serveur figée au 06/10/2026 09:00 UTC
* **Outil :** `get_member_fees` & `create_loan`
* **Observation :** Le calcul de retard donne 179 jours (26,85 €) alors que la date système réelle donne 180 ou 181 jours. De plus, `create_loan` génère un `started_at` daté du 06/10/2026 à 09:00 UTC (`1791277200`).
* **Comportement réel :** L'horloge de référence de la base de données est figée au 06/10/2026 09:00:00 UTC.
* **Règle :** Utiliser le timestamp `1791277200` comme date de référence de calcul pour concorder exactement avec la facturation du serveur.

### P8 — Schéma d'adhérent non homogène (`email: null` vs clé absente) et doublons
* **Outil :** `get_member`, `list_members`
* **Observation :** Chez les adhérents non joignables par mail, 5 ont `"email": null` tandis que 3 n'ont **pas de clé `email` du tout** dans leur JSON. Par ailleurs, `MB-200` et `MB-237` correspondent à la même personne (« Yanis Robin ») avec la même adresse mail.
* **Comportement réel :** Absence de schéma strict en base : le champ peut être nul ou absent. Des doublons d'adhérents existent.
* **Règle :** Tester `member.get("email")` avec valeur par défaut `None`. Dédupliquer les adresses lors des campagnes d'envoi.

### P9 — Réponses vides `ok: true` sous forte charge
* **Outil :** `count_books`, `list_books`, `list_loans`
* **Observation :** Lors d'une rafale d'appels, le serveur retourne parfois `{"ok": true, "count": 0}` ou une liste vide sans lever d'erreur HTTP.
* **Comportement réel :** Sous charge ou problème de session, le cache applicatif répond à vide sans avertissement.
* **Règle :** Ne jamais accepter un résultat à 0 sans recouper avec un second appel espacé de quelques secondes.

### P10 — Filtre invalide silencieux (aucun message d'erreur)
* **Outil :** `list_books`, `list_loans`
* **Observation :** Passer `status: "closed"` ou `genre: "ROMAN"` renvoie `{"ok": true, "items": []}` sans indiquer que la valeur est invalide.
* **Comportement réel :** Le backend applique un filtre SQL strict sans valider les valeurs énumérées.
* **Règle :** Vérifier que les valeurs transmises respectent scrupuleusement la casse et les valeurs documentées (`roman`, `open`...).

### P11 — Paradoxe temporel historique (`added_at` postérieur au début du prêt)
* **Outil :** `get_book`, `list_loans`
* **Observation :** 3 livres du catalogue (`BK-1065`, `BK-1075`, `BK-1070`) ont une date d'ajout `added_at` postérieure de 20 à 61 jours à la date de début de leur premier prêt (`started_at`).
* **Comportement réel :** Des données historiques incohérentes ont été injectées il y a 12 ans lors de migrations sans contraintes d'intégrité temporelle.
* **Règle :** Ne jamais rejeter un prêt comme invalide sur la seule base de la date d'ajout du livre au catalogue.
