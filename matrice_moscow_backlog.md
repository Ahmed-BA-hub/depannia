# Backlog Technique MoSCoW — DépannIA
## Découpage Agile des User Stories & Critères d'Acceptation

---

## 1. MUST HAVE (Indispensables pour la Livraison Finale S7)

| ID | User Story Technique | Critères d'Acceptation (DoD) | Séance Cible |
| :-: | :--- | :--- | :-: |
| **US-01** | **Ingestion & Chunking des notices**<br>En tant que Data Engineer, je veux découper chaque notice PDF en chunks étiquetés pour retrouver la bonne page du bon modèle. | - Script `scripts/ingest.py` avec gestion des erreurs.<br>- Recursive Character, 512 tokens, overlap 64 (comparé à 256).<br>- Métadonnées : `doc_id`, `marque`, `modele`, `type`, `page`, `chunk_id`, `source`.<br>- 100 % des notices de `data/raw/` indexées. | **S2** |
| **US-02** | **Indexation vectorielle Qdrant (HNSW)**<br>En tant que système RAG, je veux insérer les vecteurs bge-m3 dans Qdrant pour accélérer la recherche sémantique. | - Qdrant démarré via Docker Compose.<br>- Index HNSW, distance Cosine, 1024 dimensions.<br>- Insertion par lots de 64 chunks ; relancer l'ingestion ne crée pas de doublons (identifiants stables). | **S2** |
| **US-03** | **Recherche hybride & Reranking**<br>En tant que conseiller, je veux que la recherche comprenne le sens de ma question et trouve aussi les codes exacts (E:02). | - Top 20 dense + top 20 BM25, fusion RRF (k = 60).<br>- Cross-encoder `bge-reranker-base` : top 20 → top 5, en moins de 150 ms.<br>- Recall@5 mesuré $\ge 80\%$. | **S3** |
| **US-04** | **API FastAPI & Schémas Pydantic v2**<br>En tant qu'application cliente, je veux interroger l'API via des endpoints typés et validés. | - Schémas `QueryRequest` et `QueryResponse` stricts.<br>- Code HTTP 422 si la question est vide ou trop longue.<br>- Documentation OpenAPI / Swagger interactive. | **S4** |
| **US-05** | **Réponse en streaming SSE avec citations**<br>En tant que conseiller, je veux lire la réponse mot par mot, avec ses sources, pour ne pas attendre. | - Endpoint `/api/v1/chat/stream` (événements status, sources, token, done).<br>- Time-to-First-Token $< 800$ ms.<br>- Réponse « la notice ne permet pas de conclure » si le meilleur score est sous le seuil. | **S4 / S5** |
| **US-06** | **Interface web Next.js**<br>En tant que conseiller, je veux poser ma question et vérifier chaque source en un clic. | - Chat en streaming (états idle, loading, error) et bouton Stop.<br>- Pastilles [1] à [5] ouvrant le volet des sources (texte du chunk, page, score).<br>- Zéro rechargement de page. | **S5** |
| **US-07** | **Assurance Qualité & CI/CD**<br>En tant que Lead Developer, je veux que chaque commit déclenche les tests et le linter. | - Au moins 4 tests Pytest avec mocks (modèle et Qdrant), en moins de 3 s.<br>- Ruff et mypy sans erreur.<br>- GitHub Actions vert, bloquant toute fusion non conforme sur `main`. | **S6** |
| **US-08** | **Conteneurisation de Production**<br>En tant qu'ingénieur DevOps, je veux lancer tout le système avec un Docker Compose de production sécurisé. | - Dockerfiles multi-stage, utilisateur non-root, images < 350 Mo.<br>- `docker-compose.prod.yml` avec health checks.<br>- Secrets injectés uniquement par variables d'environnement. | **S7** |

---

## 2. SHOULD HAVE (Forte Valeur Ajoutée)

| ID | User Story Technique | Critères d'Acceptation | Séance Cible |
| :-: | :--- | :--- | :-: |
| **US-09** | **Évaluation Ragas automatique**<br>En tant que Data Scientist, je veux réévaluer les réponses après chaque changement. | - `data/benchmark_golden.json` : au moins 15 questions + 2 à 3 questions pièges.<br>- Faithfulness et Answer Relevance calculées par un script ; résultats dans `docs/fiche_evaluation_ia.md`. | **S3** |
| **US-10** | **Cache sémantique Redis**<br>En tant qu'architecte système, je veux servir instantanément les questions déjà posées. | - Seuil de similarité cosinus > 0.95 ; réponse servie en < 50 ms. | **S4** |

---

## 3. COULD HAVE (Options)

| ID | User Story Technique | Critères d'Acceptation |
| :-: | :--- | :--- |
| **US-11** | **Manifestes Kubernetes** | Deployments, Services et Ingress ; mots de passe dans un Secret, pas dans une ConfigMap. |
| **US-12** | **Lecture des tableaux en image des notices** | Extraction des tableaux de codes erreur présents sous forme d'image. |

---

## 4. WON'T HAVE (Exclus du Périmètre)

- Entraînement ou fine-tuning d'un modèle de langage.
- Application mobile native (l'interface web responsive suffit).
- Facturation SaaS et paiement.
- Notices scannées (images) : hors périmètre du MVP.
