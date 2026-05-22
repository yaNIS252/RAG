RAG Support
Pipeline de génération augmentée par récupération (RAG) conçu pour le support client. Construit avec LlamaIndex, Mistral AI, Qdrant et FastAPI.
🛠️ Architecture du Projet
Structure des dossiers
code
Text
.
├── src/
│   ├── config.py          # Configuration globale (Pydantic Settings - source unique de vérité)
│   ├── ingestion/         # Ingestion : Firecrawl → Découpage → Vectorisation → Import Qdrant
│   │   ├── crawler.py
│   │   ├── chunker.py
│   │   ├── embedder.py
│   │   ├── vector_store.py
│   │   └── pipeline.py
│   ├── query/             # Requête : Récupération → Reranking → Génération → Scoring → Routage
│   │   ├── retriever.py
│   │   ├── reranker.py
│   │   ├── generator.py
│   │   ├── scorer.py
│   │   ├── router.py
│   │   └── pipeline.py
│   └── api/               # API FastAPI
│       ├── main.py
│       ├── schemas.py
│       ├── dependencies.py
│       ├── middleware.py
│       └── routes/
├── scripts/               # Scripts d'entrée en ligne de commande (CLI)
│   ├── ingest.py
│   └── query.py
└── tests/                 # Tests unitaires et d'intégration
Choix techniques
Vectorisation (Embeddings) : intfloat/multilingual-e5-base (768 dimensions) pour le support multilingue (français/anglais). Le modèle est chargé une seule fois au démarrage et partagé entre les pipelines d'ingestion et de requête.
Reranking : cross-encoder/ms-marco-MiniLM-L-12-v2 (~130 Mo, ~100 ms d'exécution sur CPU).
Modèle de génération (LLM) : Mistral small-latest, offrant un bon équilibre entre coût et performance pour le support de niveau 1.
Extraction web : Firecrawl pour extraire les pages web dans un format Markdown propre.
Idempotence : Utilisation d'identifiants de fragments (chunks) déterministes basés sur UUID5 pour éviter les doublons lors des écritures dans Qdrant.
Calcul de confiance : Score combinant la pertinence (via une fonction sigmoïde), la cohérence de la réponse, la couverture des citations et la longueur du texte. Un malus de 35 % est appliqué si le modèle génère une réponse indiquant que l'information n'est pas disponible.
Limitation de débit (Rate Limiting) : Implémenté en mémoire (fenêtre glissante de 10 requêtes/seconde par clé API). Peut être remplacé par Redis pour un déploiement multi-instance.
⚙️ Installation et Configuration
1. Cloner et configurer l'environnement virtuel
code
Bash
# Création de l'environnement virtuel
python -m venv .venv

# Activation de l'environnement virtuel
# Sur Windows :
.venv\Scripts\activate
# Sur Linux/macOS :
source .venv/bin/activate

# Installation des dépendances
pip install -r requirements.txt
2. Variables d'environnement
Copiez le fichier d'exemple et renseignez vos clés API :
code
Bash
cp .env.example .env
💻 Utilisation en Ligne de Commande (CLI)
Ingestion de données
Pour indexer un site de documentation :
code
Bash
python -m scripts.ingest --client demo --url https://docs.example.com --max-pages 50 --verify
Requête directe
Pour interroger l'index via le terminal :
code
Bash
python -m scripts.query --client demo --question "Comment puis-je m'authentifier ?" --debug
🌐 API
Démarrage du serveur
code
Bash
uvicorn src.api.main:app --reload --port 8000
🔒 Sécurité : Tous les points de terminaison (excepté /v1/health) requièrent l'en-tête X-API-Key: <VOTRE_MASTER_API_KEY>.
Liste des routes principales
Méthode	Route	Description
POST	/v1/ingest	Lance une tâche d'ingestion asynchrone (retourne un task_id)
GET	/v1/ingest/{task_id}	Récupère l'état d'avancement d'une tâche d'ingestion
POST	/v1/query	Soumet une question (retourne la réponse, le score et l'action recommandée)
GET	/v1/health	Vérifie l'état de santé du service, de Qdrant et de Mistral (sans authentification)
GET	/v1/clients/{client_id}/stats	Statistiques du client (nombre de fragments, volume de requêtes)
GET	/metrics	Métriques au format Prometheus
Exemple de requête de recherche
code
Bash
curl -X POST http://localhost:8000/v1/query \
  -H "X-API-Key: $MASTER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"client_id": "demo", "question": "Quelle est la politique de remboursement ?"}'
Exemple de réponse :
code
JSON
{
  "answer_text": "...",
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
    "C1": 0.34,
    "C2": 0.28,
    "C3": 0.14,
    "C4": 0.06,
    "P": 1.0,
    "final": 0.82
  }
}
Logique de routage des actions (action)
auto_reply (Score 
≥
≥
 0.75) : Niveau de confiance élevé. La réponse peut être envoyée directement à l'utilisateur.
suggest_to_agent (
0.50
≤
0.50≤
 Score 
<
0.75
<0.75
) : Niveau de confiance modéré. Présenter la réponse à un agent humain pour validation avant envoi.
escalate (Score 
<
0.50
<0.50
) : Niveau de confiance insuffisant. Transfert direct de la demande à un conseiller.
🧪 Tests
Pour exécuter la suite de tests unitaires et d'intégration :
code
Bash
pytest tests/ -v
🔒 Sécurité et production
⚠️ Attention : L'utilisation de MASTER_API_KEY comme secret partagé unique est insuffisante pour un environnement de production multi-locataire (multi-tenant).
Avant d'ouvrir le service à des clients réels, veillez à effectuer les modifications suivantes :
Remplacer la vérification de clé en mémoire par une requête en base de données associant chaque clé API à un compte client.
Appliquer des quotas et des limitations de débit spécifiques à chaque client.
Permettre la rotation des clés d'API sans redémarrage de l'application (par exemple en stockant des empreintes de clés hachées dans PostgreSQL).
Envisager l'usage de jetons JWT à courte durée de vie avec mécanisme de rafraîchissement pour les applications clientes directes.
Ne committez jamais votre fichier .env. Utilisez les gestionnaires de secrets de votre plateforme d'hébergement.
🚀 Déploiement sur Railway
Procédure de déploiement
Installer l'interface CLI Railway :
code
Bash
npm install -g @railway/cli  # Ou via Homebrew : brew install railway
S'authentifier :
code
Bash
railway login
Initialiser ou lier le projet :
code
Bash
railway init   # Choisir "Empty project" s'il s'agit d'un premier déploiement
# Ou pour lier un projet existant :
railway link
Déployer l'application :
code
Bash
railway up
ℹ️ Le premier déploiement prend environ 10 minutes en raison du téléchargement des poids des modèles d'embedding et de reranking (~400 Mo) qui sont ensuite intégrés à l'image Docker. Les déploiements suivants bénéficient du cache de couche Docker et prennent environ 3 minutes.
Configuration des variables d'environnement sur Railway
Après le premier déploiement, rendez-vous sur votre tableau de bord Railway, puis dans l'onglet Variables de votre service pour y ajouter les clés suivantes :
Variable	Notes
MISTRAL_API_KEY	Clé d'API pour le LLM Mistral
QDRANT_URL	Point de terminaison de votre cluster Qdrant Cloud
QDRANT_API_KEY	Clé API Qdrant (sans le préfixe #)
FIRECRAWL_API_KEY	Clé d'API pour le service de crawling de pages web
MASTER_API_KEY	Générer avec : python -c "import secrets; print(secrets.token_hex(32))"
CORS_ALLOWED_ORIGINS	Domaines frontend autorisés (séparés par des virgules)
Variables optionnelles (disposant de valeurs par défaut) : EMBEDDING_MODEL, RERANKER_MODEL, MISTRAL_MODEL, SCORER_C1_SCALE_FACTOR, ROUTER_*, LOG_LEVEL.
Vérification du déploiement
Vous pouvez tester l'état du service déployé en exécutant le script de test :
code
Bash
API_URL=$(railway status --json | python -c "import sys,json; print(json.load(sys.stdin)['url'])") \
MASTER_API_KEY=<votre-cle-api> \
python scripts/smoke_test.py
Ou en interrogeant directement le point d'accès santé :
code
Bash
curl https://<votre-service-railway>.up.railway.app/v1/health
Remarques importantes pour la production
Mémoire vive : Le service requiert au minimum 2 Go de RAM (les modèles locaux occupent environ 450 Mo, le reste sert de marge pour le traitement des requêtes). Ajustez cette limite dans l'onglet Settings → Memory Limit de votre tableau de bord Railway.
Mise à l'échelle (Scaling) : Ne configurez pas plus d'une seule réplique de l'application sans utiliser d'instance Qdrant partagée ou de base de données Redis centralisée pour le gestionnaire de tâches et le limiteur de débit.
Mise en veille (Autoscale-to-zero) : Désactivez l'arrêt automatique en cas d'inactivité. Le chargement initial des modèles prend environ 30 secondes, ce qui risque de faire échouer les tests d'état de fonctionnement (healthchecks) de Railway lors des redémarrages à froid.
