RAG Support
Pipeline de Génération Augmentée par Récupération (RAG) prêt pour la production, conçu pour automatiser et assister le support client de niveau 1.
Ce projet est construit sur une architecture moderne utilisant LlamaIndex, Mistral AI, Qdrant et FastAPI.
💡 Comprendre le RAG (Retrieval-Augmented Generation)
Quel est le problème avec les LLM classiques ?
Un grand modèle de langage (LLM) comme Mistral ou GPT est entraîné sur un volume gigantesque de données publiques à un instant T. Il présente deux limites majeures pour un cas d'usage professionnel :
L'absence de connaissances privées : Le LLM ne connaît pas vos procédures internes, votre politique de remboursement ou les détails de votre API.
L'hallucination : Face à une question dont il ignore la réponse, un LLM a tendance à inventer une réponse plausible mais factuellement incorrecte.
La Solution : Le RAG
Le RAG résout ces problèmes en agissant comme un moteur de recherche intelligent couplé à un rédacteur professionnel. Au lieu de demander au LLM de chercher la réponse dans sa "mémoire" (ses poids), le pipeline procède en trois étapes :
code
Text
1. RECHERCHE (Retrieve)          2. ENRICHISSEMENT (Augment)      3. GÉNÉRATION (Generate)
 ┌──────────────────────┐         ┌─────────────────────────┐      ┌──────────────────────┐
 │ Question de l'usager │ ──────► │ Question + Documents    │ ───► │ Réponse fiable       │
 │ + Base vectorielle   │         │ pertinents (Contexte)   │      │ rédigée avec sources │
 └──────────────────────┘         └─────────────────────────┘      └──────────────────────┘
La Récupération (Retrieve) : Dès qu'une question est posée (ex: "Comment être remboursé ?"), le système interroge une base de données vectorielle (Qdrant) pour y trouver les passages de votre documentation les plus pertinents.
L'Augmentation (Augment) : On construit un "prompt" (une consigne) enrichi de la question initiale et des extraits de documents trouvés.
La Génération (Generate) : Le LLM (Mistral) lit ces extraits et rédige une réponse claire et factuelle. Il a pour consigne stricte de refuser de répondre s'il ne trouve pas l'information dans le contexte fourni, éliminant ainsi presque totalement les hallucinations.
🏗️ Architecture du Projet
Le système sépare strictement la phase de préparation des données (Ingestion) et la phase d'interaction (Requête/RAG).
code
Text
┌────────────────────────────────────────┐
               │          API REST (FastAPI)            │
               └───────────────────┬────────────────────┘
                                   │
         ┌─────────────────────────┴─────────────────────────┐
         ▼                                                   ▼
┌─────────────────────────────────┐                 ┌─────────────────────────────────┐
│     1. Pipeline d'Ingestion     │                 │      2. Pipeline de Requête     │
│   (Préparation des données)     │                 │      (Moteur de Recherche RAG)  │
└─────────────────────────────────┘                 └─────────────────────────────────┘
1. Le Pipeline d'Ingestion (Asynchrone)
Ce pipeline transforme des pages web brutes en "connaissances" assimilables par la base de données vectorielle.
code
Text
[ URL ] ➔ ( Firecrawl ) ➔ [ Markdown Propre ] ➔ ( Chunker ) ➔ [ Fragments ]
                                                                    │
 [ Qdrant ] ◄── ( Import Idempotent ) ◄── [ Vecteurs (768d) ] ◄── ( Multi-E5 )
Extraction (Firecrawl) : Télécharge les pages d'un site web et élimine le bruit (menus, footers) pour ne garder qu'un format Markdown propre.
Découpage (Chunking) : Découpe le texte en blocs (chunks) de taille homogène pour que le modèle puisse les analyser précisément.
Idempotence (UUID5) : Assigne un identifiant unique à chaque fragment basé sur son contenu textuel. Cela évite les doublons dans la base si le script est lancé plusieurs fois.
Vectorisation (Embedding - E5) : Transforme chaque bloc de texte en une suite de 768 nombres représentant son sens sémantique (vecteur).
Base Vectorielle (Qdrant) : Stocke ces vecteurs et les textes associés pour permettre des recherches ultra-rapides.
2. Le Pipeline de Requête (RAG Synchrone)
Ce pipeline s'exécute en quelques secondes lorsqu'un utilisateur pose une question.
code
Text
[ Question ] ➔ ( Qdrant Retriever ) ➔ [ Top-K Fragments ] ➔ ( MiniLM Reranker ) ➔ [ Contexte Filtré ]
                                                                                          │
 [ Routage ] ◄── ( Confidence Scorer ) ◄── [ Réponse + Sources ] ◄── ( Mistral LLM ) ◄────┘
Récupération (Retrieval) : Convertit la question en vecteur et récupère les blocs de texte sémantiquement proches dans Qdrant.
Réordonnancement (Reranking - MiniLM) : Évalue de manière plus fine la pertinence de chaque fragment par rapport à la question. Les fragments jugés hors-sujet sont rejetés pour ne conserver que le contexte indispensable.
Synthèse (Generation - Mistral) : Mistral formule une réponse rédigée en s'appuyant uniquement sur les documents retenus et liste les sources utilisées.
Calcul de Confiance (Scoring) : Un algorithme évalue la qualité de la réponse (présence de citations, cohérence, détection de phrases d'évitement comme "Je ne sais pas"). Un score entre 0 et 1 est attribué.
Aiguillage (Routing) : Détermine si la réponse est assez fiable pour être envoyée directement au client ou si elle doit être relue par un humain.
📁 Structure des Fichiers
code
Text
.
├── src/
│   ├── config.py          # Configuration globale (Pydantic - variables d'environnement)
│   ├── ingestion/         # Pipeline d'importation des connaissances
│   │   ├── crawler.py     # Récupération web via Firecrawl
│   │   ├── chunker.py     # Découpage du texte en blocs
│   │   ├── embedder.py    # Modèle de vectorisation (E5)
│   │   ├── vector_store.py# Connexion et opérations Qdrant
│   │   └── pipeline.py    # Orchestration de l'ingestion
│   ├── query/             # Pipeline d'exécution RAG
│   │   ├── retriever.py   # Recherche dans Qdrant
│   │   ├── reranker.py    # Réordonnancement via MiniLM
│   │   ├── generator.py   # Appel et instructions LLM Mistral
│   │   ├── scorer.py      # Calcul du score de confiance
│   │   ├── router.py      # Logique de routage de la réponse
│   │   └── pipeline.py    # Orchestration de la requête
│   └── api/               # Couche d'exposition HTTP (FastAPI)
│       ├── main.py        # Point d'entrée de l'API
│       ├── schemas.py     # Modèles de données (validation Pydantic)
│       ├── dependencies.py# Injection de dépendances et sécurité
│       ├── middleware.py  # Limiteur de débit, logs, CORS
│       └── routes/        # Déclaration des endpoints
├── scripts/               # Points d'accès CLI (Terminal)
│   ├── ingest.py          # Script d'ingestion manuel
│   └── query.py           # Script de test de requête manuel
└── tests/                 # Tests automatisés
🛠️ Installation et Configuration
Prerequis
Python 3.10 ou supérieur
1. Cloner et configurer l'environnement
code
Bash
# Créer l'environnement virtuel
python -m venv .venv

# Activer l'environnement virtuel
# Sur Windows :
.venv\Scripts\activate
# Sur Linux/macOS :
source .venv/bin/activate

# Installer les dépendances
pip install -r requirements.txt
2. Configuration des variables d'environnement
Créez un fichier .env à la racine à partir du modèle d'exemple :
code
Bash
cp .env.example .env
Renseignez vos clés API dans le fichier .env :
MISTRAL_API_KEY : Clé d'accès à l'API Mistral AI.
QDRANT_URL & QDRANT_API_KEY : Vos accès à l'instance de base de données vectorielle.
FIRECRAWL_API_KEY : Pour le scraping des sites web de documentation.
MASTER_API_KEY : Clé secrète d'accès à votre API FastAPI (à générer).
🚀 Utilisation
En Ligne de Commande (CLI)
Étape 1 : Indexer une documentation
Téléchargez et découpez un site web pour l'injecter dans la base vectorielle.
code
Bash
python -m scripts.ingest --client demo --url https://docs.example.com --max-pages 50 --verify
Étape 2 : Interroger la base en direct
Testez le pipeline de recherche et de génération directement depuis votre terminal.
code
Bash
python -m scripts.query --client demo --question "Comment puis-je m'authentifier ?" --debug
Via l'API HTTP (Production)
Démarrez le serveur FastAPI localement :
code
Bash
uvicorn src.api.main:app --reload --port 8000
Toutes les requêtes (sauf /v1/health) nécessitent l'en-tête X-API-Key: <VOTRE_MASTER_API_KEY>.
Exemple de requête RAG (Interrogation)
code
Bash
curl -X POST http://localhost:8000/v1/query \
  -H "X-API-Key: VotreCleSecrete" \
  -H "Content-Type: application/json" \
  -d '{"client_id": "demo", "question": "Quelle est la politique de remboursement ?"}'
Exemple de réponse retournée :
code
JSON
{
  "answer_text": "Notre politique permet un remboursement intégral sous 14 jours...",
  "confidence_score": 0.82,
  "action": "auto_reply",
  "sources": ["https://docs.example.com/refunds"],
  "latency_ms": {
    "retrieve_ms": 50,
    "rerank_ms": 100,
    "generate_ms": 2100,
    "total_ms": 2260
  },
  "cost_estimate_eur": 0.000345,
  "score_breakdown": {
    "C1": 0.34, "C2": 0.28, "C3": 0.14, "C4": 0.06, "P": 1.0, "final": 0.82
  }
}
Comprendre les actions de routage automatique (action)
L'API calcule une décision d'aiguillage selon le score de confiance obtenu :
auto_reply (Score 
≥
≥
 0.75) : Réponse très fiable. Peut être envoyée directement au client final sans intervention humaine.
suggest_to_agent (
0.50
≤
0.50≤
 Score 
<
0.75
<0.75
) : Réponse incertaine. Présentée à un agent humain pour relecture ou ajustement avant envoi.
escalate (Score 
<
0.50
<0.50
) : Confiance trop faible. La demande est directement envoyée à un agent sans proposition de réponse automatisée.
🔒 Sécurité et passage à l'échelle
⚠️ Limites du MVP : La clé MASTER_API_KEY actuelle est un secret partagé statique. C'est insuffisant pour du multi-tenant en production.
Avant d'ouvrir l'API à des clients externes :
Gestion des clés : Remplacez la vérification statique par un appel en base de données associant chaque clé API à un client unique.
Quota et Rate Limiting : Remplacez le limiteur de débit en mémoire par une instance Redis partagée pour gérer le multi-instance.
Rotation des clés : Stockez les clés sous forme hachée en base de données (type PostgreSQL) pour permettre leur révocation en temps réel sans coupure de service.
🚀 Déploiement sur Railway
Étapes rapides de mise en ligne
S'authentifier sur le CLI Railway :
code
Bash
railway login
Lier votre dossier au projet Railway :
code
Bash
railway link
Lancer le build et déployer :
code
Bash
railway up
Recommandations d'infrastructure
Mémoire : Configurez votre instance Railway avec au moins 2 Go de RAM (le chargement initial des modèles d'embedding et de reranking nécessite environ 450 Mo).
Mise en veille : Désactivez l'option d'arrêt automatique en cas d'inactivité (Autoscale-to-zero). Le chargement des modèles au démarrage prend environ 30 secondes, ce qui déclencherait des erreurs de timeout sur les requêtes à froid.
