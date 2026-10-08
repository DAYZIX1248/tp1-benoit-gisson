# TP 1 — Faire parler une API que personne ne documente

**Binôme :** Benoit & Gisson  
**Agent utilisé :** Antigravity IDE  
**Configuration de rendu :** `opencode.json` (conforme au standard d'évaluation)  
**Serveur MCP :** `bibliotheque-municipale` (`https://ai-tools-mcp-biblio-production.up.railway.app/mcp`)  

---

## Gestion du token

Le token personnel n'est commité dans **aucun fichier suivi par Git** :
- `opencode.json` référence `{env:MCP_TOKEN}` : la valeur est injectée dynamiquement depuis l'environnement.
- `mcp_config.json` et `antigravity.json` référencent `${MCP_TOKEN}` pour la compatibilité avec l'IDE.
- La clé réelle réside exclusivement dans le fichier local `.env`, formellement ignoré par `.gitignore`.
- Un modèle [`.env.example`](.env.example) est fourni pour documenter le format sans divulguer le secret.
- Vérification effectuée : `git log -p | grep biblio-` ne retourne aucun résultat dans l'historique Git.

---

## Exercice 1 — Intégrer le serveur MCP

### 1a. Liste des outils exposés
Connexion établie sur le transport HTTP streamable (status `200`, `serverInfo.name = "bibliotheque-municipale"`, version `1.0.0`). Le serveur expose **12 outils** :

| Domaine | Outils exposés |
| :--- | :--- |
| **Livres** | `list_books`, `count_books`, `get_book`, `search_books` |
| **Adhérents** | `list_members`, `get_member` |
| **Prêts** | `list_loans`, `get_member_fees`, `create_loan`, `delete_loan` |
| **Missions** | `list_missions`, `get_mission` |

![Liste des outils exposés](screenshots/01_tools_list.png)

### 1b. Réponse brute d'un outil de lecture
Appel direct de `get_book({"book_id": "BK-1042"})` retournant les vraies données de la médiathèque :

```json
{
  "ok": true,
  "book": {
    "book_id": "BK-1042",
    "title": "Le Dernier de verre",
    "author": "Karim Barbier",
    "genre": "policier",
    "copies": 4,
    "loan_duration": 21,
    "added_at": "2023-11-09T09:00:00.000Z",
    "archived": false
  }
}
```

![Réponse brute d'un outil de lecture](screenshots/02_raw_tool_read.png)

### 1c. Énoncé exact des cinq missions
Extraits fidèlement du champ `brief` via `list_missions` et `get_mission` (M1 à M5) :

1. **M1 — Inventaire :** « Le conseil municipal demande le nombre exact d'ouvrages détenus par la bibliothèque, et la répartition par genre. Donne les chiffres et explique comment tu les as obtenus. »
2. **M2 — Le retardataire :** « Identifie l'emprunt le plus en retard actuellement : quel adhérent, quel ouvrage, et combien de jours de retard exactement. Donne aussi le montant dû par cet adhérent. »
3. **M3 — La réinscription :** « Enregistre un nouvel emprunt pour l'adhérent MB-214 sur l'ouvrage BK-1042, puis vérifie que l'emprunt apparaît bien dans sa fiche. »
4. **M4 — Le ménage :** « L'adhérent MB-202 demande l'effacement de ses emprunts déjà rendus. Supprime-les, puis prouve qu’ils ont bien disparu. »
5. **M5 — La relance :** « Prépare la campagne de relance : la liste des adhérents ayant au moins un emprunt en retard et joignables par mail, et le nombre de ceux qui ne sont pas joignables, en distinguant les cas. »

---

## Exercice 2 — Mener les cinq missions (Journal de bord)

Chaque mission a été exécutée et recoupée contre les données brutes paginées de l'ensemble du registre.

### 2.1 Déroulement des missions

#### Mission 1 — Inventaire
* **Appel initial :** `count_books({})` $\rightarrow$ `{"ok": true, "count": 184}`
* **Ce qui clochait :** 184 inclut les 26 ouvrages archivés. `list_books` sans option ne liste que 158 livres en service. De plus, `limit` est silencieusement plafonné à 50 (P2), et `next` continue indéfiniment après la fin (P1).
* **Réponse retenue :** **158 titres en service (415 exemplaires physiques)**. Total général : **184 titres / 490 exemplaires** en comptant les 26 titres archivés (75 exemplaires).  
  *Répartition en service (titres / exemplaires) :* roman 30/71, policier 27/79, jeunesse 30/82, essai 23/65, bd 19/48, poésie 29/70.  
* **Tentatives :** 1 tentative avec recoupement complet.

#### Mission 2 — Le retardataire
* **Appel initial :** `list_loans({"status": "open", "limit": 50})` paginé sur les 53 emprunts ouverts.
* **Ce qui clochait :** L'emprunt le plus ancien `LN-5106` est situé en page 3. L'outil `get_member_fees` renvoie `overdue_duration: 4296` qui est en **heures** (P6). L'horloge du serveur est figée au 06/10/2026 à 09:00 UTC (P7), donnant **179 jours** (26,85 €) alors que la date système réelle donne 180 ou 181 jours.
* **Réponse retenue :** Prêt **LN-5106**, **Paul Blanc (MB-225)**, ouvrage *Le Retour des autres* (BK-1075), échéance au 10/04/2026 09:00 UTC $\rightarrow$ **179 jours de retard** selon l'horloge figée du serveur (180 jours au calendrier réel) ; **26,85 €** dus (179 j × 0,15 €).
* **Tentatives :** 1 tentative.

#### Mission 3 — La réinscription
* **Appels :** `create_loan({"member_id": "MB-214", "book_id": "BK-1042"})` $\rightarrow$ `{"ok": false, "error": "missing field"}`.
* **Ce qui clochait :** Le paramètre obligatoire `desk_code` est absent du schéma MCP (P4). De plus, l'emprunt n'apparaît pas dans `get_member` (qui n'a pas de champ de prêts), mais doit être vérifié dans `list_loans(member_id="MB-214")`.
* **Deuxième appel :** `create_loan({"member_id": "MB-214", "book_id": "BK-1042", "desk_code": "A1"})` $\rightarrow$ `LN-5137` créé avec succès (date de début figée au 06/10/2026 09:00 UTC selon P7).
* **Réponse retenue :** Prêt **LN-5137** enregistré au guichet A1 pour Chloé Roux (MB-214), visible dans `list_loans`.
* **Tentatives :** 2 tentatives.

#### Mission 4 — Le ménage
* **Appels :** Identification des 7 prêts de MB-202 (6 rendus et 1 ouvert `LN-5060`). Suppression unitaire des 6 rendus via `delete_loan`.
* **Ce qui clochait :** `delete_loan` répond `{"ok": true, "deleted": true}` mais n'efface rien : il effectue un soft-delete en basculant `archived: true` (P5).
* **Preuve :** `list_loans(member_id="MB-202", include_archived=false)` montre que les prêts rendus ont disparu du registre actif (seul LN-5060 reste). Mais `list_loans(member_id="MB-202", include_archived=true)` prouve qu'ils subsistent avec `archived: true`.
* **Réponse retenue :** Les 6 prêts rendus (LN-5038, LN-5039, LN-5062, LN-5095, LN-5120, LN-5134) sont **archivés mais non effacés physiquement**. LN-5060 reste actif.
* **Tentatives :** 1 tentative (non dupe du soft-delete).

#### Mission 5 — La relance
* **Appels :** Pagination des 53 prêts ouverts $\rightarrow$ 49 prêts en retard concernant **29 adhérents distincts**.
* **Ce qui clochait :** Schéma d'adhérent non homogène (P8) : certains ont `"email": null`, d'autres n'ont **pas de clé `email`** dans le JSON. De plus, `MB-200` et `MB-237` sont des homonymes parfaits avec la même adresse (« Yanis Robin », doublon).
* **Réponse retenue :**  
  - **29 adhérents** en retard au total.  
  - **Joignables par email : 21 adhérents** (représentant **20 adresses distinctes** suite au doublon MB-200 / MB-237) : MB-200, 201, 202, 204, 207, 208, 210, 214, 216, 217, 219 (adhésion inactive), 221, 225, 227, 228, 230, 231, 237, 239, 240, 242.  
  - **Non joignables : 8 adhérents**, ventilés selon deux cas techniques :  
    1. *Clé présente avec valeur nulle (`"email": null`) :* 5 adhérents (MB-203, MB-212, MB-226, MB-232, MB-234).  
    2. *Clé `email` totalement absente du JSON :* 3 adhérents (MB-206, MB-235, MB-241).  
* **Tentatives :** 1 tentative.

---

### 2b. L'agent s'est déclaré satisfait d'un résultat faux (Cas observés)

1. **Fausse conclusion sur la pagination :** Lors de la pagination filtrée de M3, voyant des pages vides successives, l'agent a conclu : *« Le serveur applique le filtre après la pagination, donc la plupart des pages reviennent vides »*. **C'est faux** : le filtre est bien appliqué en amont en base de données. Les pages vides résultent uniquement du curseur `next` qui ne s'arrête jamais après la fin (piège P1).
2. **Incohérence des retards dans M2 :** Sans précaution, l'agent calcule le retard à partir de la date système réelle (180 ou 181 jours) tout en annonçant un montant dû de 26,85 €. Or 180 × 0,15 € = 27,00 €. L'agent présente ces deux chiffres contradictoires avec assurance, sans comprendre que le serveur utilise une horloge figée au 06/10/2026 à 09:00 UTC (P7).
3. **Erreur masquée sur `create_loan` :** L'agent a cru que l'adhérent n'existait pas ou que le livre était indisponible face au message `{"ok": false, "error": "missing field"}`, avant de découvrir qu'il s'agissait du paramètre caché `desk_code`.

![Cas d'erreur masquée et faux positif](screenshots/03_agent_false_positive.png)

---

## Exercice 3 — Le skill

### 3.1 Démarche et outillage
Le skill a été rédigé selon le standard officiel **Agent Skills** (`SKILL.md` avec frontmatter YAML `name` et `description`).
- **Outil retenu :** Génération structurée conforme à la spécification Anthropic/OpenCode/Antigravity.
- **Pourquoi ce choix :** Ce format garantit la **divulgation progressive** (*progressive disclosure*) : l'IDE n'injecte que la description au repos pour préserver le contexte, et ne charge l'intégralité des 11 règles que lorsqu'une tâche manipule la médiathèque.
- **Emplacement :** `.agents/skills/bibliotheque-api/SKILL.md`.

### 3.2 Catalogue officiel des 11 pièges (P1 à P11)

Conformément à la consigne « *Tout signaler ne paie pas* », les comportements expressément décrits dans les schémas (archives exclues par défaut dans `list_books`, `memberId` en camelCase, valeurs de filtres en minuscules) ont été classés en **comportements normaux** et non en anomalies.

| Identifiant | Piège avéré | Type d'anomalie |
| :--- | :--- | :--- |
| **P1** | `next` jamais `null` après la fin de la collection (pages vides sans fin) | Non-terminaison |
| **P2** | `limit` plafonné silencieusement à 50 sans avertissement | Plafonnement masqué |
| **P3** | `count_books` (184) ≠ `list_books` (158) : archivés comptés en douce | Biais d'inventaire |
| **P4** | `create_loan` exige `desk_code`, non documenté dans l'inputSchema | Paramètre fantôme |
| **P5** | `delete_loan` répond `deleted: true` mais effectue un soft-delete | Sémantique mensongère |
| **P6** | `get_member_fees` : unités horaires, centimes et cumul global non documentés | Unités absconses |
| **P7** | Horloge serveur figée au 06/10/2026 à 09:00 UTC | ✔ *Trouvaille inattendue* |
| **P8** | `email: null` vs clé absente, doublon d'adresse (MB-200 / MB-237) | Incohérence schéma |
| **P9** | Réponses vides `ok: true` sous forte charge ou perte de session | ✔ *Trouvaille inattendue* |
| **P10** | Filtre invalide retournant une liste vide sans message d'erreur | Erreur silencieuse |
| **P11** | 3 livres dont le prêt commence avant la date d'ajout (`added_at`) | ✔ *Trouvaille inattendue* |

*Détail de P11 (Paradoxe temporel historique) :*
- Prêt `LN-5061` sur `BK-1065` (*Un Été de Marseille*) : début du prêt le 08/05/2026 alors que le livre a été ajouté le 28/05/2026 (+20 jours).
- Prêt `LN-5106` sur `BK-1075` (*Le Retour des autres*) : début du prêt le 27/03/2026 alors que le livre a été ajouté le 18/04/2026 (+22 jours).
- Prêt `LN-5108` sur `BK-1070` (*Le Jardin du fleuve*) : début du prêt le 28/04/2026 alors que le livre a été ajouté le 28/06/2026 (+61 jours).

---

### 3.3 Validation et preuves d'efficacité

#### a. Comparatif Avant / Après sur les missions
Avec le skill chargé, l'agent évite immédiatement les pièges :

![Comparatif Mission 1 avant et après le skill](screenshots/04_before_after_m1.png)

![Comparatif Mission 2 avant et après le skill](screenshots/05_before_after_m2.png)

- **Mission 1 :** L'agent s'arrête à la page < 50 sans boucler indéfiniment (P1/P2) et distingue immédiatement les 158 titres actifs des 26 archivés.
- **Mission 2 :** L'agent identifie `LN-5106`, applique l'horloge figée (179 jours) et convertit les heures et centimes sans contradiction.
- **Mission 3 :** `desk_code: "A1"` est fourni dès le premier essai : réussite immédiate sans échec `missing field`.
- **Mission 4 :** L'agent vérifie immédiatement avec `include_archived: true` et répond avec lucidité que les prêts sont archivés et non effacés.
- **Mission 5 :** Traitement direct des clés absentes et déduplication des adresses.

#### b. Preuve de chargement du Skill par l'agent
Le skill est détecté et injecté dynamiquement dans la session dès que la tâche mentionne la médiathèque :

![Chargement du skill par l'agent](screenshots/06_skill_loaded.png)

#### c. Pourquoi un skill et pas une commande ? (Trois lignes)
> Une **commande** est un prompt impératif ou un script déterministe que l'utilisateur doit déclencher manuellement (`/commande`) pour exécuter une tâche figée sans capacité d'adaptation.  
> Un **skill** est une compétence procédurale et heuristique que l'agent mobilise de façon autonome dès que le contexte de sa tâche correspond à la description du skill.  
> À l'usage, le skill permet à l'agent d'appliquer les contournements de l'API sur n'importe quelle requête imprévue, là où une commande ne saurait que répéter une procédure rigide.
