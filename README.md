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
│           └── SKILL.md              # Le Skill documentant les 11 pièges et règles de l'API
├── screenshots/                      # Captures d'écran probantes exigées par le sujet
│   ├── 01_tools_list.png             # Découverte des 12 outils exposés
│   ├── 02_raw_tool_read.png          # Appel de lecture avec réponse brute JSON
│   ├── 03_agent_false_positive.png   # Exemple d'erreur masquée et faux résultat
│   ├── 04_before_after_m1.png        # Comparatif Avant / Après sur la Mission 1
│   ├── 05_before_after_m2.png        # Comparatif Avant / Après sur la Mission 2
│   └── 06_skill_loaded.png           # Chargement du skill par Antigravity IDE
├── mcp_config.json                   # Configuration MCP Antigravity (sans token en clair)
├── antigravity.json                  # Déclaration MCP IDE
├── .gitignore                        # Protection contre les fuites de secrets (.env)
├── .env.example                      # Gabarit de configuration d'environnement
└── RAPPORT.md                        # Rapport complet d'exercice et journal de bord
```

---

## 🎯 Résultats Clés des 5 Missions

| Mission | Intitulé | Résultat Final Validé |
| :--- | :--- | :--- |
| **M1** | **Inventaire** | **158 titres actifs** en circulation (**415 exemplaires physiques**). Fonds total avec archivés : 184 titres (490 exemplaires). |
| **M2** | **Le retardataire** | Emprunt le plus en retard : **`LN-5106`** (Adhérent : **`MB-225`** Paul Blanc, Livre : **`BK-1075`**). Retard : **179 jours exacts** (4 296 h). Montant dû : **26,85 €**. |
| **M3** | **La réinscription** | Emprunt créé avec succès avec le paramètre caché guichet (`desk_code: "A1"`). Vérifié dans `list_loans(member_id="MB-214")`. |
| **M4** | **Le ménage** | Les 6 emprunts déjà rendus de **`MB-202`** ont été archivés. Registre actif ramené à 0 emprunt rendu. Traces archivées auditées. |
| **M5** | **La relance** | **29 adhérents en retard** au total : **20 joignables par email** (actifs avec email valide) et **9 non joignables** (8 avec email manquant et 1 adhérente avec compte inactif). |

---

## 🛡️ Résumé des 11 Pièges Documentés dans le Skill

1. **`create_loan` :** Paramètre obligatoire caché `desk_code` (absent du schéma MCP).
2. **`delete_loan` :** Mensonge systématique : valide `ok: true` même sur des identifiants inexistants.
3. **`delete_loan` :** Soft-delete masqué : bascule simplement `archived: true` au lieu de supprimer.
4. **`delete_loan` & `get_member_fees` :** Effacement frauduleux des dettes financières sur suppression d'un prêt en cours.
5. **`get_member_fees` :** Unités horaires (non documentées) et cumul global sur tous les prêts de l'adhérent.
6. **`count_books` :** Inventaire trompeur incluant les 26 archivés et aveugle à tous les filtres.
7. **`create_loan` :** Absence de contrôle de statut (accepte les livres archivés et membres inactifs).
8. **`create_loan` :** Overbooking illimité (dépassement libre du stock physique `copies`).
9. **`search_books` :** Renvoie silencieusement des archives et applique une sensibilité stricte aux accents.
10. **`list_loans` :** Jeton `next` trompeur sur collection vide (risque de boucle infinie).
11. **Tous outils :** Incohérence temporelle tripartite (mélange de `DD/MM/YYYY`, ISO 8601 et UNIX timestamps).

---

## 🚀 Consultation

Consultez [**`RAPPORT.md`**](RAPPORT.md) pour le détail méthodologique et les réponses aux questions académiques.
