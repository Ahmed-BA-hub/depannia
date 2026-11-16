# Choix de la base vectorielle : Qdrant

## Notre besoin

- Chaque question porte sur un appareil précis : la recherche doit rester dans la notice de ce modèle, grâce à un filtre sur les métadonnées « marque » et « modèle ».
- Volume : environ 2 000 chunks aujourd'hui, environ 150 000 en production, avec des vecteurs bge-m3 de 1 024 dimensions.
- Le système doit se lancer en local, simplement, avec Docker Compose.

## Comparaison sur les 3 critères du cours

| Critère | Qdrant | Milvus | pgvector |
| :--- | :--- | :--- | :--- |
| **Filtrage par métadonnées** | Le filtre s'applique pendant le parcours de l'index HNSW, grâce à un index sur les métadonnées : on obtient bien les 20 résultats demandés, tous du bon modèle. | Le filtre (expression booléenne) s'applique pendant la recherche : bon. | Avec un index HNSW, le filtre `WHERE` s'applique après le parcours de l'index : avec un filtre strict (une notice sur 30), on peut obtenir moins de 20 résultats. Depuis la version 0.8, un « iterative scan » optionnel réduit ce problème, mais il faut le régler. |
| **Latence** | Moteur dédié écrit en Rust, index HNSW : quelques millisecondes attendues à notre échelle. | Moteur dédié, index HNSW : quelques millisecondes attendues à notre échelle. | Index HNSW dans PostgreSQL : rapide à notre échelle, mais le filtrage strict le pénalise. |
| **Simplicité d'orchestration locale** | Un seul conteneur, avec un tableau de bord web intégré pour voir les chunks. | Le Docker Compose officiel de Milvus standalone lance 3 conteneurs (Milvus, etcd, MinIO). | Un seul conteneur PostgreSQL avec l'extension. |

## Décision

Nous retenons **Qdrant** :

1. il filtre par modèle d'appareil **pendant** la recherche, ce qui est indispensable pour nous, puisque toutes nos questions portent sur une notice précise ;
2. il tient dans **un seul conteneur**, lancé par notre `docker-compose.yml` ;
3. son **tableau de bord** permet de montrer les chunks et leurs métadonnées pendant les audits.

Milvus est écarté car il est trop lourd pour notre volume (3 conteneurs). pgvector est écarté à cause du filtrage après la recherche, qui peut rendre moins de résultats que demandé.

La latence de recherche de Qdrant sera mesurée en séance 2 sur nos questions de référence.
