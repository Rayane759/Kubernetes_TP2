# TP Helm : Packager le backend avec PostgreSQL (CloudNativePG)

## Objectifs du TP

- Comprendre les concepts fondamentaux de Helm (charts, templates, values)
- Créer un chart Helm pour le backend d'une application
- Utiliser CloudNativePG (CNPG) comme dépendance pour la base de données
- Maîtriser les opérations de mise à jour et rollback

Livrables :

- Un chart Helm déployant le backend et PostgreSQL (via CNPG)
- Un compte avec les difficultés rencontrées et les résultats obtenus.

## Prérequis

- Avoir un cluster Kubernetes opérationnel (SSPCloud)
- Helm installé  (`helm version` doit fonctionner)
- kubectl configuré pour accéder à votre cluster
- Avoir réalisé le TP 3 (déploiement de l'application 3-tier)

> **Important** : Avant de commencer, nettoyez les ressources des TP précédents si vous en avez encore :
>
> ```bash
> kubectl delete deployment backend --ignore-not-found
> kubectl delete service backend --ignore-not-found
> ```

## Exercice 1 : Prise en main de Helm et CloudNativePG

### 1.1 Installation et vérification

1. Vérifier que Helm est installé : `helm version`
2. Ajouter le repository CloudNativePG :
   ```bash
   helm repo add cnpg https://cloudnative-pg.github.io/charts
   helm repo update
   ```
3. Lister les charts disponibles dans ce repo : `helm search repo cnpg`

### 1.2 Exploration du chart cluster CNPG

1. Deux charts apparaissent dans le repo CNPG :
   - `cnpg/cloudnative-pg` : l'opérateur Kubernetes (installé au niveau cluster)
   - `cnpg/cluster` : un chart pour déployer un cluster PostgreSQL géré par CNPG

2. Afficher les valeurs par défaut du chart `cnpg/cluster` :
   ```bash
   helm show values cnpg/cluster
   ```


NB : CNPG utilise un pattern opérateur avec des CRD (Custom Resource Definition). L'idée de ce type d'architecture est de faciliter le déploiement et la gestion d'outils spécifiques (ici une postegres). Sur le SSP Cloud, l'opérateur est déjà installé, ce qui permettra d'installer des clusters postgres facilement. 


## Exercice 2 : Un chart pour le Backend

L'objectif est de créer un chart Helm pour le backend. Le chart ne contiendra que le backend dans un premier temps ; la connexion à PostgreSQL sera ajoutée à l'exercice suivant.

### 2.1 Création du squelette

1. Créer un nouveau chart : `helm create backend-app`
2. Explorer la structure générée :

   ```
   backend-app/
   ├── Chart.yaml          # Métadonnées du chart
   ├── values.yaml         # Valeurs par défaut
   ├── templates/          # Templates Kubernetes
   │   ├── deployment.yaml
   │   ├── service.yaml
   │   ├── ingress.yaml
   │   ├── _helpers.tpl    # Fonctions utilitaires
   │   └── ...
   └── charts/             # Dépendances
   ```

3. **Supprimer les fichiers inutiles** dans `templates/` : `hpa.yaml`, `serviceaccount.yaml`, `tests/`, `ingress.yaml`. On gardera `deployment.yaml`, `service.yaml`, `_helpers.tpl` et `NOTES.txt`.

### 2.2 Développement du Chart

1. Modifier `Chart.yaml` :

```yaml
apiVersion: v2
name: backend-app
description: Chart Helm pour le backend de l'application 3-tier
type: application
version: 0.1.0
appVersion: "1.1"
```

2. Analyser le fichier `values.yaml` afin d'identifier ce qui est déjà disponible et les éléments manquants. 

3. Écrire  le template pour le deployment : `templates/deployment.yaml` et pour le service `templates/deployment.yaml`. 

L'objectif est d'avoir quelque chose de réutilisable pour d'autres applications (backend, frontend). Par conséquent, il est nécessaire d'utiliser correctement la syntaxe de templating Go. 

Il est possible qu'il n'y ait pas beaucoup de changements dans les fichiers créés par `helm create`.

4. Éditer le fichier `values.yaml`


### 2.3 Validation et premier déploiement

1. Valider le chart : `helm lint ./backend-app`
2. Visualiser les manifests générés : `helm template ma-release ./backend-app`
3. Installer le chart : `helm install ma-release ./backend-app`
4. Vérifier le déploiement : `kubectl get deployments,svc,pods`
5. Le backend démarre mais ne peut pas se connecter à PostgreSQL (normal à ce stade). Vérifiez les logs :
   ```bash
   kubectl logs deployment/ma-release-backend-app-backend
   ```


## Exercice 3 : Intégration de PostgreSQL via CNPG

> **Important** : Avant de commencer, nettoyez les ressources des TP précédents si vous en avez encore :
>
> ```bash
> kubectl delete statefulset postgresql --ignore-not-found
> kubectl delete service postgresql --ignore-not-found
> ```



On va maintenant ajouter CloudNativePG comme **dépendance** du chart, pour que Helm gère l'ensemble backend + base de données en une seule release.

### 3.1 Déclaration de la dépendance

1. Modifier `Chart.yaml` pour ajouter la dépendance CNPG :

```yaml
apiVersion: v2
name: backend-app
description: Chart Helm pour le backend de l'application 3-tier
type: application
version: 0.2.0
appVersion: "1.1"

dependencies:
  - name: cluster
    version: "0.0.9"       # Vérifiez la version disponible avec : helm search repo cnpg/cluster
    repository: "https://cloudnative-pg.github.io/charts"
    alias: postgresql
```

> L'alias `postgresql` permet de configurer cette dépendance via la clé `postgresql:` dans `values.yaml`.

2. Télécharger la dépendance :
   ```bash
   helm dependency update ./backend-app
   ```

3. Observer le dossier `charts/` : un fichier `.tgz` a été téléchargé. Observer aussi le fichier `Chart.lock` généré.

### 3.2 Convention de nommage CNPG

Quand CNPG crée un cluster PostgreSQL via ce chart, il génère automatiquement :
- Une ressource `Cluster` nommée `<release>-cluster`
- Des **services** pour accéder au cluster
- Un **secret** contenant les credentials de connexion

Pour votre release `ma-release`, voici les ressources créées :

| Ressource | Nom |
|-----------|-----|
| Service primary (lecture/écriture) | `ma-release-postgresql-rw` |
| Service read-only | `ma-release-postgresql-r` |
| Secret des credentials | `ma-release-postgresql-app` |

Le secret `ma-release-postgresql-app` contient les clés `username` et `password`, prêtes à être utilisées dans le backend via `secretKeyRef`.

### 3.3 Mise à jour du chart

1. Mettre à jour `values.yaml` pour configurer la dépendance CNPG en ajoutant la section suivante :

```yaml
# Configuration du cluster PostgreSQL (dépendance CNPG)
postgresql:
  cluster:
    instances: 1
    storage:
      size: "1Gi"
    initdb:
      database: reviews
      owner: admin
```

2. Mettre à jour `templates/deployment.yaml` pour lire les credentials depuis le secret CNPG :

```yaml
apiVersion: apps/v1
kind: Deployment
...
        env:
        - name: DB
          value: {{ .Values.env.DB | quote }}
        - name: DB_HOST
          value: {{ printf "%s-postgresql-rw" .Release.Name | quote }}
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: {{ printf "%s-postgresql-app" .Release.Name }}
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: {{ printf "%s-postgresql-app" .Release.Name }}
              key: password
```

   > On utilise `secretKeyRef` pour injecter les credentials directement depuis le secret créé par CNPG, sans jamais exposer les mots de passe dans les templates ou dans `values.yaml`.

### 3.4 Déploiement complet

1. Mettre à jour la release :
   ```bash
   helm upgrade ma-release ./backend-app
   ```

2. Vérifier que tous les composants sont déployés :
   ```bash
   kubectl get deployments,clusters,svc,pods
   ```

3. Observer la ressource `Cluster` créée par CNPG :
   ```bash
   kubectl describe cluster ma-release-postgresql
   ```

4. Attendre que le cluster PostgreSQL soit prêt (quelques minutes) et vérifier les secrets créés :
   ```bash
   kubectl get secrets | grep postgresql
   ```
kubectl logs deployment/ma-release-backend-app-backend
5. Une fois PostgreSQL prêt, vérifier les logs du backend :
   ```bash
   
   ```

## Exercice 4 : Mise à jour et rollback

### 4.1 Mise à jour

1. Modifier le `values.yaml` pour passer le backend à 2 réplicas
2. Mettre à jour : `helm upgrade ma-release ./backend-app`
3. Vérifier que 2 pods backend sont en cours d'exécution : `kubectl get pods`
4. Consulter l'historique des releases : `helm history ma-release`

### 4.2 Rollback

1. Revenir à la version précédente : `helm rollback ma-release 2`
2. Vérifier que le nombre de réplicas est revenu à 1
3. Consulter à nouveau l'historique : `helm history ma-release`


### 4.3 Surcharge de valeurs

1. Créer un fichier `values-prod.yaml` en passant le nombre de replicas à 3. 

2. Déployer avec ces valeurs : `helm upgrade ma-release ./backend-app -f values-prod.yaml`
3. Vérifier : `kubectl get pods` et `kubectl get cluster`


## Nettoyage

```bash
helm uninstall ma-release
kubectl get all  # Vérifier que toutes les ressources ont été supprimées
```

## Bonus : chart Helm 3tier

L'objectif de ce dernier exercice est d'ajouter le frontend dans le chart Helm. 

L'idée est d'installer toute la stack avec une unique ligne de commande. 