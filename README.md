# TP 1 — Faire parler une API que personne ne documente

**Binôme :** Benoit & Gisson  
**Projet :** `tp1-benoit-gisson`  
**Environnement :** Antigravity IDE  
**Serveur MCP :** `bibliotheque-municipale` (`https://ai-tools-mcp-biblio-production.up.railway.app/mcp`)  

---

## 📌 Vue d'ensemble

Ce projet contient l'ensemble du travail d'exploration, d'audit critique, de résolution des 5 missions et de capitalisation sous forme de **Skill** pour le serveur MCP de la médiathèque municipale.

Le compte rendu complet et détaillé est disponible dans [**`RAPPORT.md`**](RAPPORT.md).

---

## 📂 Structure du Dépôt

```text
tp1-benoit-gisson/
├── .agents/
│   └── skills/
│       └── bibliotheque-api/
│           └── SKILL.md              # Le Skill documentant les 11 pièges réels (P1 à P11)
├── screenshots/                      # Captures d'écran probantes exigées par le sujet
│   ├── 01_tools_list.png             # Découverte des 12 outils exposés
│   ├── 02_raw_tool_read.png          # Appel de lecture avec réponse brute JSON
│   ├── 03_agent_false_positive.png   # Exemple d'erreur masquée et faux résultat
│   ├── 04_before_after_m1.png        # Comparatif Avant / Après sur la Mission 1
│   ├── 05_before_after_m2.png        # Comparatif Avant / Après sur la Mission 2
│   └── 06_skill_loaded.png           # Chargement du skill par l'agent
├── opencode.json                     # Configuration officielle OpenCode demandée
├── mcp_config.json                   # Configuration MCP Antigravity
├── antigravity.json                  # Déclaration MCP IDE
├── .gitignore                        # Protection contre les fuites de secrets (.env)
├── .env.example                      # Gabarit de configuration d'environnement
└── RAPPORT.md                        # Rapport complet d'exercice et journal de bord
```

---

## 🎯 Résultats des Cinq Missions

| Mission | Intitulé | Résultat Final Retenu |
| :--- | :--- | :--- |
| **M1** | **Inventaire** | **158 titres en service** (415 exemplaires physiques). Total catalogue avec 26 archivés : 184 titres (490 exemplaires). |
| **M2** | **Le retardataire** | Prêt **`LN-5106`** (Paul Blanc / MB-225, livre `BK-1075`). Retard : **179 jours** selon l'horloge figée du serveur (180 j calendrier réel). Montant dû : **26,85 €**. |
| **M3** | **La réinscription** | Prêt `LN-5137` enregistré au guichet `A1` pour Chloé Roux (MB-214) sur `BK-1042`. Vérifié dans `list_loans`. |
| **M4** | **Le ménage** | Les 6 prêts rendus de MB-202 sont **archivés mais non effacés physiquement** (soft-delete). LN-5060 reste actif. |
| **M5** | **La relance** | **29 adhérents en retard**. **21 joignables** (20 adresses distinctes suite au doublon Yanis Robin). **8 non joignables** (5 avec `email: null` et 3 avec clé `email` absente). |

---

## 🛡️ Les 11 Pièges Réels (P1 à P11)

| Id | Piège | Nature |
| :--- | :--- | :--- |
| **P1** | Curseur `next` jamais `null` après la fin | Non-terminaison de pagination |
| **P2** | Paramètre `limit` plafonné à 50 sans avertissement | Plafonnement silencieux |
| **P3** | `count_books` (184) ≠ `list_books` (158) | Biais d'inventaire |
| **P4** | `create_loan` exige `desk_code` hors schéma | Paramètre fantôme |
| **P5** | `delete_loan` répond `deleted: true` mais effectue un soft-delete | Sémantique trompeuse |
| **P6** | `get_member_fees` : heures / centimes / cumul non documentés | Unités absconses |
| **P7** | Horloge serveur figée au 06/10/2026 09:00 UTC | ✔ *Trouvaille inattendue* |
| **P8** | `email: null` vs clé absente, doublon d'adhérent (MB-200 / MB-237) | Incohérence schéma |
| **P9** | Réponses vides `ok: true` sous charge | ✔ *Trouvaille inattendue* |
| **P10** | Filtre invalide retournant une liste vide sans erreur | Erreur silencieuse |
| **P11** | 3 prêts dont le début est antérieur à la date d'ajout du livre | ✔ *Trouvaille inattendue* |
