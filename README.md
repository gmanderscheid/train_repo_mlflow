# Continuous Training — 100% GitHub Actions

Démo pédagogique : un pipeline de **continuous training** entièrement piloté par GitHub Actions, sans orchestrateur externe (pas d'Airflow ici).

## Le flow

```
GitHub Actions (workflow_dispatch ou cron)
        │
        ├─ JOB 1 : docker build ──> docker push ──> Docker Hub
        │
        └─ JOB 2 : aws ec2 run-instances (t3.medium éphémère)
                        │
                        │  user-data au démarrage :
                        │    1. installe Docker
                        │    2. pull l'image de train depuis Docker Hub
                        │    3. lance l'entraînement dans le conteneur
                        │    4. le modèle + métriques partent sur MLflow
                        │       (artefacts sur S3, backend Postgres)
                        │    5. shutdown -h now ──> l'instance se TERMINE seule
                        │
                        └─ le job attend la terminaison = entraînement fini ✅
```

Point clé pour les élèves : **la machine d'entraînement n'existe que le temps de l'entraînement**. On paye quelques minutes de t3.medium, puis plus rien. Le flag `--instance-initiated-shutdown-behavior terminate` transforme le `shutdown` interne en terminaison AWS complète.

## Structure du repo

```
train-repo/
├── train/
│   ├── train.py            # pipeline attrition IBM (RF + GridSearchCV) + MLflow
│   ├── requirements.txt
│   └── Dockerfile
└── .github/workflows/
    └── continuous-training.yml
```

## Prérequis

1. Un **serveur MLflow** accessible publiquement (backend store Postgres, artifact store S3)
2. Un compte **Docker Hub** avec un access token (Account Settings → Security)
3. Un **utilisateur IAM** AWS avec les droits EC2 (`RunInstances`, `DescribeInstances`, `CreateTags`), SSM `GetParameter` (lecture de l'AMI publique) et écriture sur le bucket S3 des artefacts MLflow

## Secrets GitHub à créer

Dans le repo : Settings → Secrets and variables → Actions → New repository secret

| Secret | Contenu |
|---|---|
| `DOCKERHUB_USERNAME` | votre username Docker Hub |
| `DOCKERHUB_TOKEN` | access token Docker Hub |
| `AWS_ACCESS_KEY_ID` | clé IAM |
| `AWS_SECRET_ACCESS_KEY` | secret IAM |
| `AWS_DEFAULT_REGION` | ex. `eu-west-3` |
| `MLFLOW_TRACKING_URI` | URL de votre serveur MLflow, ex. `https://mon-mlflow.example.com` |

## Lancer un entraînement

Onglet **Actions** → workflow *Continuous Training* → **Run workflow**. Deux inputs :

- `tune` : `true` = grid search complet (12 combinaisons × 5 folds, optimisé F1), `false` = fit rapide avec hyperparamètres fixes (pratique en démo live)
- `register_alias` : alias posé sur la nouvelle version dans le registry (`challenger` par défaut — l'API de prod, elle, sert l'alias `champion`)

Puis vérifiez dans l'UI MLflow : expérience `ibm_attrition_detector`, un run `ec2_rf_tuned` avec `test_f1`, `test_roc_auc`, la matrice de confusion et le `classification_report.txt` en artefacts, et une nouvelle version du modèle `ibm_attrition_detector` portant l'alias choisi.

Pour un réentraînement planifié, décommentez le bloc `schedule` dans le workflow.

## Tester l'image en local avant de tout brancher

```bash
cd train
docker build -t train-demo .
docker run --rm \
  -e MLFLOW_TRACKING_URI="https://mon-mlflow.example.com" \
  -e AWS_ACCESS_KEY_ID="..." \
  -e AWS_SECRET_ACCESS_KEY="..." \
  -e AWS_DEFAULT_REGION="eu-west-3" \
  -e TUNE="false" \
  train-demo
```

## Limites assumées (et pistes pour aller plus loin)

- **Les secrets transitent par le user-data** : le user-data est lisible via les métadonnées de l'instance. Acceptable pour une démo sur une instance éphémère qu'on contrôle ; en production on passerait par un **IAM instance profile** (plus aucune clé AWS à injecter) et **SSM Parameter Store / Secrets Manager** pour l'URI MLflow.
- **Pas de logs de l'entraînement dans GitHub Actions** : le job ne voit que l'état de l'instance. En production : envoyer les logs du conteneur vers CloudWatch, ou piloter l'exécution via SSM `send-command` pour récupérer stdout.
- **t3.medium** (2 vCPU / 4 Go) absorbe le grid search sur le dataset IBM (~1 470 lignes) en quelques minutes ; adapter `--instance-type` pour un dataset plus lourd.
- L'instance part dans le **VPC par défaut** avec le security group par défaut : aucun port entrant nécessaire, seul du trafic sortant (Docker Hub, MLflow, S3) est utilisé.
