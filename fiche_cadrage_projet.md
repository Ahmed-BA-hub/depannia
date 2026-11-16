# Fiche de Cadrage d'Ingénierie — Projet Fédérateur AI-BD_BI
## Spécification du Moteur RAG : DépannIA

---

### 1. Fiche d'Identité du Projet

- **Nom du Projet :** `DépannIA`
- **Axe :** RAG hybride (recherche dense + BM25, reranking cross-encoder)
- **Membres de l'Équipe :**
  1. *Ahmed — Rôles : Lead Data / Retrieval, Lead Backend / MLOps, Lead Frontend / Infra*
- **Lien du Dépôt GitHub :** `https://github.com/Ahmed-BA-hub/depannia`
- **Branche de travail principale :** `main` (protégée par la CI à partir de la séance 6)

---

### 2. Cadrage du Problème Métier ("Problem-First")

#### 2.1 Persona Cible & Contexte Opérationnel
- **Utilisateur visé :** conseiller du service après-vente d'une enseigne d'électroménager (persona : Sami), qui répond aux appels des clients.
- **Point de douleur majeur (*Pain point*) :** pour chaque appel (« mon lave-vaisselle affiche E:02 »), le conseiller doit retrouver la notice du bon modèle parmi des centaines de PDF, puis la bonne page : environ 5 à 10 minutes par appel (estimation). Pendant ce temps, le client attend, et une mauvaise réponse peut entraîner le déplacement inutile d'un technicien.
- **Limite des approches classiques (Pourquoi l'IA générative ?) :**
  - la recherche par mots-clés (Ctrl+F, Elasticsearch) échoue quand le client et la notice n'emploient pas les mêmes mots (« le lave-vaisselle reste plein d'eau » / « défaut de vidange ») ;
  - la recherche par le sens seule confond les codes proches (E:01 / E:02) ;
  - un même code n'a pas le même sens d'une marque à l'autre : la recherche doit rester dans la notice du bon modèle.

#### 2.2 Proposition de Valeur
Une interface de discussion en streaming qui donne, en quelques secondes, la solution tirée de la notice du bon modèle, en 5 étapes maximum, chaque étape citant sa page. Elle indique si la notice propose une solution que le client peut appliquer lui-même ou si elle renvoie vers un technicien, et elle répond « la notice ne permet pas de conclure » quand l'information n'y est pas.

---

### 3. Données & Volumétrie

- **Format des données sources :** `[x] PDF multi-colonnes` `[ ] JSON/CSV` `[ ] Tables relationnelles SQL` `[ ] Markdown` `[ ] Images/Scans`
  - notices d'utilisation officielles des fabricants, en français, au format PDF texte (pas de scans), avec des tableaux de codes erreur.
- **Volumétrie initiale (PoC S1-S3) :** `environ 30 notices / environ 2 000 chunks (estimation)` — 4 types d'appareils (lave-linge, lave-vaisselle, réfrigérateur, four), 4 à 5 marques.
- **Volumétrie cible en production :** `environ 2 000 notices / environ 150 000 chunks (projection pour une enseigne)`
- **Fréquence de mise à jour des données :** `[ ] Batch quotidien` `[ ] Temps réel / Streaming` `[x] Statique` (ajout par lot quand une nouvelle notice arrive)
- **Métadonnées de chaque chunk :** marque, modèle, type d'appareil, fichier, page.

---

### 4. Architecture Technique & Composants RAG

- **Stratégie de Chunking :** `[x] Recursive Character` `[ ] Semantic Chunking` `[ ] Parent-Document / Hierarchical`
  - *Taille de chunk visée :* `512 tokens` | *Overlap :* `64 tokens`
  - découpage page par page, pour que chaque chunk garde sa page ; la taille 512 sera comparée à 256 en séance 2 (Recall@5 sur 10 questions).
- **Modèle d'Embeddings :** `BAAI/bge-m3` (Dimension: `1024`) — multilingue, très bon en français, gratuit, exécuté en local via Ollama.
- **Base Vectorielle retenue :** `[x] Qdrant` `[ ] Milvus` `[ ] pgvector`
  - *Justification du choix :* `Qdrant filtre par métadonnées (modèle d'appareil) pendant la recherche HNSW, tient dans un seul conteneur Docker et offre un tableau de bord intégré. pgvector filtre par défaut après la recherche (moins de résultats possibles) ; Milvus demande plusieurs conteneurs.`
- **Recherche hybride :** vecteurs denses (top 20) + BM25 (top 20), fusion RRF (k = 60) → top 20.
- **Reranker (Cross-Encoder) :** `BAAI/bge-reranker-base` (servi par Text Embeddings Inference) — top 20 → top 5.
- **LLM de Génération :** `Gemma 3 4B via Ollama, en local` — gratuit, les données restent locales ; appel compatible OpenAI pour basculer vers un modèle hébergé en changeant un seul paramètre.
- **Application :** API FastAPI asynchrone + Pydantic v2, streaming SSE, interface Next.js avec citations cliquables, historique dans PostgreSQL 16.

---

### 5. Métriques Quantitatives & SLAs

| Métrique | Définition / Outil | Objectif Cible (KPI) |
| :--- | :--- | :---: |
| **Recall@5 (Retrieval)** | Le chunk de vérité est parmi les 5 chunks transmis au modèle | $\ge 80\%$ |
| **Faithfulness (Ragas)** | Part des affirmations de la réponse appuyées par les chunks | $\ge 0.85$ / 1.00 |
| **Answer Relevance (Ragas)** | Adéquation directe à la question posée | $\ge 0.80$ / 1.00 |
| **Latence First-Token (TTFT)** | Temps d'affichage du premier token en streaming | $< 800$ ms |
| **Latence E2E P95** | Temps total de complétion, pour 95 % des questions | $< 3.5$ s |

- *Pourquoi Recall@5 :* le modèle ne lit que les 5 chunks gardés par le reranker.
- *Mesure :* Ragas et chronométrage sur un jeu de référence d'au moins 15 questions écrites et vérifiées à la main (`data/benchmark_golden.json`), plus 2 à 3 questions pièges hors notices, auxquelles le système doit refuser de répondre.

---

### 6. Gestion des Risques & Garde-fous

- **Prévention des Hallucinations :** Injection systématique de la directive : *"Si l'information n'est pas présente dans les extraits ci-dessus, répondez explicitement que le corpus ne permet pas de conclure."* De plus, si le meilleur score du reranker est sous un seuil, le système répond « la notice ne permet pas de conclure » sans appeler le modèle ; les citations [1] à [5] sont rattachées par le code, pas écrites par le modèle.
- **Protection des Secrets :** Utilisation exclusive de variables d'environnement via `.env`, ignoré par Git (`.gitignore` créé avant le premier commit) ; seul `.env.example` est versionné.
- **Isolation Réseau :** Qdrant et PostgreSQL confinés dans un réseau Docker interne privé ; seuls l'interface et l'API sont exposées.
- **Qualité des données :** PDF texte en français uniquement ; extraction du texte vérifiée sur 5 notices avant l'indexation complète.
- **Latence :** modèle local léger ; plan B : modèle hébergé via la même interface compatible OpenAI.
