---
name: bibliotheque-api
description: >-
  Guide opérationnel et contournement des pièges de l'API MCP "bibliotheque-municipale".
  À activer impérativement avant toute manipulation du catalogue, des adhérents ou des emprunts.
---

# Guide Opérationnel — API Serveur MCP Médiathèque Municipale

Ce skill récapitule les comportements réels, incohérences de schéma et pièges non documentés du serveur MCP de la médiathèque municipale. Tout agent interagissant avec cette API doit appliquer rigoureusement les règles ci-dessous pour garantir l'exactitude de ses résultats.

---

## 1. Outil : `count_books` & `list_books` (Inventaire et volumes)

* **Ce que l'on observe :**  
  L'appel à `count_books` retourne une valeur fixe (184). Si l'on somme les livres retournés par `list_books` (sans paramètre), on n'obtient que 158 ouvrages. L'agent naïf se contredit entre le total annoncé et la liste des genres.
* **Ce que fait réellement le serveur :**  
  `count_books` compte tous les titres enregistrés en base, y compris les 26 ouvrages archivés (`archived: true`, retirés de la circulation). À l'inverse, `list_books` filtre silencieusement par défaut les archivés sauf si `include_archived: true` est explicitement passé. De plus, `count_books` et le nombre d'items de `list_books` ne comptent que les *titres* de catalogue, et ignorent le volume réel d'exemplaires physiques indiqués dans la propriété `copies` de chaque livre (415 exemplaires pour les actifs, 490 au total avec archivés).
* **Règle à appliquer :**  
  Ne jamais utiliser `count_books` pour un inventaire d'exploitation courante sans préciser qu'il inclut le fond archivé. Pour tout inventaire ou répartition par genre :
  1. Parcourir `list_books` avec pagination.
  2. Spécifier si le décompte porte sur les titres uniques en circulation (158 titres) ou le total physique d'exemplaires (`sum(copies)` = 415 exemplaires).
  3. Si l'inventaire demande le fond total détenu par la médiathèque, activer `include_archived: true` (184 titres, 490 exemplaires physiques).

---

## 2. Outils : `list_books`, `list_loans`, `list_members` (Pagination tronquée)

* **Ce que l'on observe :**  
  Les appels à ces outils ne renvoient que 20 éléments par défaut. L'agent naïf s'arrête à la première page et conclut à tort que l'ensemble des données a été inspecté.
* **Ce que fait réellement le serveur :**  
  Une pagination serveur avec limite par défaut à 20 items est appliquée. Un curseur encodé en base64 est fourni sous la clé `"next"`. Les données critiques (par exemple l'emprunt le plus ancien `LN-5106` ou les adhérents au-delà de 20) sont situées dans les pages ultérieures.
* **Règle à appliquer :**  
  Toujours boucler sur l'outil tant que `"next"` est présent et non nul dans la réponse JSON, en passant `"start_key": next_value`. Ne jamais formuler de conclusion globale sans avoir épuisé la pagination.

---

## 3. Outil : `get_member` vs Autres outils (Incohérence de casse des paramètres)

* **Ce que l'on observe :**  
  L'appel à `get_member(member_id="MB-...")` échoue avec l'erreur `{"ok": false, "error": "invalid request"}`.
* **Ce que fait réellement le serveur :**  
  Le serveur utilise une convention hybride. Tous les outils (`list_loans`, `get_member_fees`, `create_loan`) utilisent le paramètre `member_id` en `snake_case`, mais `get_member` exige strictement `memberId` en `camelCase`.
* **Règle à appliquer :**  
  Toujours invoquer `get_member` avec le paramètre exact `memberId` (camelCase) :
  `get_member(memberId="MB-XXX")`.

---

## 4. Outil : `get_member_fees` (Unité et cumul des retards)

* **Ce que l'on observe :**  
  Le champ `overdue_duration` renvoie de grands nombres (ex: 4296, 7536). L'agent a tendance à interpréter ce nombre comme des jours ou des minutes.
* **Ce que fait réellement le serveur :**  
  `overdue_duration` est exprimé en **heures** (4296 h = 179 jours). De surcroît, cette valeur est la **somme cumulée** de tous les emprunts en retard de cet adhérent, et non la durée de retard d'un emprunt particulier. Le tarif journalier `late_fee_per_day` (15) est en centimes d'euro par jour (soit 0,15 € / jour).
* **Règle à appliquer :**  
  1. Pour connaître le retard précis d'un emprunt individuel, calculer `(date_courante - due_at) / 86400` directement à partir du timestamp UNIX (en secondes) de l'emprunt.
  2. Diviser `overdue_duration` par 24 pour convertir les heures globales en jours.
  3. Ne jamais attribuer l'`overdue_duration` cumulée d'un adhérent à un seul de ses prêts si l'adhérent a plusieurs prêts ouverts.

---

## 5. Outil : `create_loan` (Paramètre obligatoire caché hors schéma)

* **Ce que l'on observe :**  
  Appeler `create_loan(member_id="...", book_id="...")` renvoie `{"ok": false, "error": "missing field"}`, bien que l'inputSchema MCP n'indique que ces deux champs comme requis.
* **Ce que fait réellement le serveur :**  
  Le backend applicatif exige obligatoirement le code guichet `desk_code` (ex: `"A1"`, `"B2"`), qui a été omis dans la déclaration du schéma MCP.
* **Règle à appliquer :**  
  Toujours inclure `desk_code` lors de la création d'un emprunt :  
  `create_loan(member_id="MB-...", book_id="BK-...", desk_code="A1")`.

---

## 6. Outil : `delete_loan` (Faux delete et absence de garde-fous)

* **Ce que l'on observe :**  
  `delete_loan` renvoie `{"ok": true, "deleted": true}`. Cependant, l'emprunt réapparaît si l'on appelle `list_loans(include_archived=True)`. De plus, le serveur accepte de "supprimer" un emprunt non rendu (`status: "open"`).
* **Ce que fait réellement le serveur :**  
  `delete_loan` n'effectue aucun hard-delete en base de données : il réalise un **soft-delete** en basculant le champ `archived` à `true`. Comme `list_loans` filtre par défaut les éléments archivés, l'emprunt semble avoir disparu, mais l'historique subsiste. De plus, le serveur ne vérifie pas si l'emprunt a été restitué.
* **Règle à appliquer :**  
  1. Avant d'appeler `delete_loan`, toujours vérifier que `status === "returned"`. Ne jamais appeler `delete_loan` sur un statut `"open"`.
  2. Pour prouver la suppression, démontrer que l'emprunt n'apparaît plus dans `list_loans(member_id="...")` mais qu'il est tracé avec `archived: true` dans `list_loans(member_id="...", include_archived=True)`.

---

## 7. Outil : `list_members` & `get_member` (Adhérents joignables vs inactifs)

* **Ce que l'on observe :**  
  Certains adhérents possèdent une adresse e-mail mais ne doivent pas être relancés, tandis que d'autres n'ont aucune adresse e-mail renseignée.
* **Ce que fait réellement le serveur :**  
  Le champ `email` peut valoir `null` / vide pour des adhérents actifs. À l'inverse, des adhérents ayant un e-mail valide peuvent avoir `active: false` (adhésion résiliée ou suspendue).
* **Règle à appliquer :**  
  Pour toute campagne de relance, croiser impérativement deux critères :
  - Joignable : `active === true` ET `email !== null` (et format e-mail valide).
  - Non joignable : distinguer formellement le cas `email manquant (null)` chez un adhérent actif du cas `adhérent inactif` (`active: false`).
