# TP 1 — Faire parler une API que personne ne documente

**Binôme :** Benoit-Gisson  
**Projet :** `tp1-benoit-gisson`  
**Environnement :** Antigravity IDE  

---

## Vue d'ensemble

Ce projet contient l'ensemble du travail d'exploration, d'audit critique et de capitalisation sous forme de **Skill** pour le serveur MCP de la médiathèque municipale (`https://ai-tools-mcp-biblio-production.up.railway.app/mcp`).

Le compte rendu complet et détaillé est disponible dans [**`RAPPORT.md`**](RAPPORT.md).

---

## Structure du Dépôt

```text
tp1-benoit-gisson/
├── .agents/
│   └── skills/
│       └── bibliotheque-api/
│           └── SKILL.md              # Le Skill documentant les 7 pièges et règles de l'API
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

## Installation & Exécution locale

1. **Configurer l'environnement :**
   ```bash
   cp .env.example .env
   # Renseigner votre token personnel dans .env (jamais commité)
   ```

2. **Consulter le rapport :**
   Ouvrez [`RAPPORT.md`](RAPPORT.md) pour retrouver tous les résultats chiffrés, le journal de bord et les règles de contournement.
