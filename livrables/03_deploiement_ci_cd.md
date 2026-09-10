# Déploiement automatisé

> Compétence évaluée : `C31` — Mettre en œuvre un système de déploiement automatisé respectant les bonnes pratiques DevOps.

**Application support :** OpsTrack Field Service

## 1. Stratégie de déploiement

### 1.1 Vue d'ensemble

Le déploiement de l'application **OpsTrack Field Service** est automatisé à l'aide de **GitHub Actions**.

Le code source est hébergé sur GitHub et versionné avec Git. La chaîne CI/CD permet de contrôler une version avant de la déployer sur les environnements de qualification et de production.

Deux environnements sont utilisés :

* **Qualification :** `eval-dfs-q-tpl-20265-08.it-students.fr`
* **Production :** `eval-dfs-p-tpl-20265-08.it-students.fr`

Le pipeline suit le principe suivant :

1. récupération du code ;
2. installation des dépendances ;
3. exécution des contrôles automatisés ;
4. déploiement sur l'environnement de qualification ;
5. vérification de la qualification avec un smoke test ;
6. promotion de la version validée vers la production ;
7. exécution d'un smoke test sur la production.

La qualification constitue ainsi un point de contrôle avant la mise en production.

Une erreur détectée pendant les tests ou pendant le déploiement en qualification empêche la poursuite du pipeline et donc la modification de la production.

Le déploiement de production peut également être protégé par une validation manuelle via un environnement GitHub Actions dédié à la production.

### 1.2 Diagramme du pipeline

```text
                         ┌───────────────────┐
                         │    Développeur    │
                         └─────────┬─────────┘
                                   │
                              git push
                                   │
                                   ▼
                         ┌───────────────────┐
                         │      GitHub       │
                         │    Repository     │
                         └─────────┬─────────┘
                                   │
                                   ▼
                    ┌──────────────────────────┐
                    │      GitHub Actions      │
                    │                          │
                    │  Checkout                │
                    │  Composer / npm          │
                    │  Tests                   │
                    │  Contrôles               │
                    └────────────┬─────────────┘
                                 │
                            Tests OK
                                 │
                                 ▼
                    ┌──────────────────────────┐
                    │       QUALIFICATION      │
                    │                          │
                    │  Déploiement              │
                    │  Migration                 │
                    │  Cache                    │
                    │  Smoke test               │
                    └────────────┬─────────────┘
                                 │
                           Smoke test OK
                                 │
                                 ▼
                    ┌──────────────────────────┐
                    │ Validation production    │
                    │       (si activée)       │
                    └────────────┬─────────────┘
                                 │
                                 ▼
                    ┌──────────────────────────┐
                    │       PRODUCTION         │
                    │                          │
                    │  Déploiement              │
                    │  Migration                 │
                    │  Cache                    │
                    │  Smoke test               │
                    └────────────┬─────────────┘
                                 │
                         ┌───────┴────────┐
                         │                │
                       Succès            Échec
                         │                │
                         ▼                ▼
                    Version validée   Diagnostic /
                                      rollback
```

---

## 2. Outillage retenu

| Outil                      | Rôle dans le pipeline                   | Justification                                                                      |
| -------------------------- | --------------------------------------- | ---------------------------------------------------------------------------------- |
| **Git**                    | Versionnement du code                   | Permet d'identifier précisément chaque version déployée                            |
| **GitHub**                 | Hébergement du dépôt                    | Déjà utilisé pour le projet et intégré avec GitHub Actions                         |
| **GitHub Actions**         | Orchestration du CI/CD                  | Permet d'automatiser les tests et les déploiements                                 |
| **SSH**                    | Connexion aux serveurs                  | Permet une communication sécurisée avec les VPS                                    |
| **Composer**               | Installation des dépendances PHP        | Garantit une installation reproductible avec `composer.lock`                       |
| **npm**                    | Installation des dépendances JavaScript | Utilisé pour le dashboard Next.js                                                  |
| **PHPUnit / Laravel Test** | Tests automatisés                       | Permet de détecter les régressions avant déploiement                               |
| **Laravel Artisan**        | Maintenance de l'application            | Permet notamment d'exécuter les migrations et de reconstruire les caches           |
| **curl**                   | Smoke tests                             | Permet de vérifier automatiquement la disponibilité HTTP/HTTPS                     |
| **Bash**                   | Script de déploiement serveur           | Permet de regrouper les opérations de déploiement dans une procédure reproductible |

Le pipeline est volontairement basé sur des outils courants afin de limiter la complexité de maintenance.

Le script `scripts/deploy.sh` centralise les opérations réalisées sur le serveur. Le workflow GitHub Actions se charge principalement de l'orchestration.

---

## 3. Déclenchement du déploiement

### 3.1 Mode de déclenchement

Le pipeline peut être déclenché automatiquement lors d'un `push` sur la branche principale.

Un déclenchement manuel est également prévu grâce à `workflow_dispatch`, notamment pour pouvoir relancer un déploiement sans effectuer de nouvelle modification du code.

Exemple :

```yaml
on:
  push:
    branches:
      - main
  workflow_dispatch:
```

Le principe de promotion est le suivant :

```text
Push sur main
      │
      ▼
Tests et contrôles
      │
      ├── Échec → arrêt
      │
      ▼
Déploiement qualification
      │
      ├── Échec → arrêt
      │
      ▼
Smoke test qualification
      │
      ├── Échec → arrêt
      │
      ▼
Promotion production
      │
      ▼
Smoke test production
```

La production n'est donc pas mise à jour si les étapes précédentes échouent.

Lorsque cela est nécessaire, une validation manuelle peut être ajoutée avant l'étape de production avec un environnement GitHub Actions protégé.

### 3.2 Reproductibilité

Le pipeline est entièrement défini dans un fichier versionné du dépôt :

```text
.github/workflows/deploy.yml
```

Les opérations exécutées sur le serveur sont regroupées dans :

```text
scripts/deploy.sh
```

Le déploiement s'appuie sur la version exacte du code présente dans Git.

La récupération de la version peut être réalisée avec :

```bash
git fetch origin
git reset --hard origin/main
```

Cette méthode permet d'éviter qu'un serveur conserve accidentellement des modifications locales et garantit que son contenu correspond à la branche déployée.

Les dépendances PHP sont installées avec :

```bash
composer install --no-dev --optimize-autoloader
```

Le fichier `composer.lock` permet de conserver les versions précises des dépendances utilisées.

Le dashboard Next.js utilise également son fichier de verrouillage des dépendances afin de garantir une installation reproductible.

Les migrations de base de données sont versionnées dans le dépôt et exécutées avec :

```bash
php artisan migrate --force
```

L'utilisation de `migrate` permet de conserver les données existantes contrairement à `migrate:fresh`, qui supprimerait les tables.

---

## 4. Contrôles préalables au déploiement

Les contrôles sont exécutés avant toute modification des serveurs.

| Contrôle              | Description                              | Critère de passage        |
| --------------------- | ---------------------------------------- | ------------------------- |
| Installation Composer | Installation des dépendances PHP         | Installation sans erreur  |
| Tests Laravel         | Exécution de la suite de tests           | Tous les tests passent    |
| Syntaxe PHP           | Vérification des fichiers PHP critiques  | Aucune erreur de syntaxe  |
| Installation npm      | Installation des dépendances Next.js     | Installation sans erreur  |
| Build Next.js         | Compilation du dashboard                 | Build terminé avec succès |
| Vérification du dépôt | Contrôle de la version Git               | Commit disponible         |
| Connexion SSH         | Vérification de l'accès au serveur cible | Connexion réussie         |

Exemple de contrôles :

```bash
composer install --no-interaction --prefer-dist
php artisan test
```

Pour le dashboard Next.js :

```bash
npm ci
npm run build
```

Une vérification syntaxique peut également être effectuée sur les fichiers PHP importants :

```bash
php -l public/hooks.php
```

Si l'un des contrôles échoue, le workflow s'arrête et aucun déploiement ne doit être effectué.

---

## 5. Mise à jour de la production

Le déploiement est réalisé par GitHub Actions au moyen d'une connexion SSH sécurisée.

La clé privée utilisée par GitHub Actions est stockée dans les secrets du dépôt GitHub et n'est pas présente dans le code source.

Le script `scripts/deploy.sh` est exécuté sur le serveur.

La procédure de mise à jour suit les étapes suivantes :

1. vérification de l'état du dépôt ;
2. mémorisation du commit actuellement déployé ;
3. récupération de la nouvelle version ;
4. installation des dépendances PHP de production ;
5. exécution des migrations ;
6. reconstruction des caches Laravel ;
7. construction ou mise à jour du dashboard Next.js ;
8. redémarrage des services nécessaires ;
9. exécution du smoke test.

Le script mémorise la version précédente afin de faciliter un retour arrière en cas de problème.

La récupération du code est réalisée avec :

```bash
git fetch origin
git reset --hard origin/main
```

Les dépendances Laravel sont ensuite installées :

```bash
composer install --no-dev --optimize-autoloader
```

Les migrations sont exécutées avec :

```bash
php artisan migrate --force
```

Les caches sont reconstruits :

```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

Les opérations propres au dashboard Next.js sont exécutées selon son mode de fonctionnement en production.

L'ensemble de ces opérations est regroupé dans le script de déploiement afin d'éviter de dépendre d'une succession de commandes manuelles.

---

## 6. Vérification post-déploiement

### 6.1 Smoke tests

Après chaque déploiement, un test de fumée est réalisé afin de vérifier que l'application est toujours accessible.

Le test principal consiste à effectuer une requête HTTPS vers l'application :

```bash
curl -f -s -o /dev/null \
  https://eval-dfs-p-tpl-20265-08.it-students.fr
```

L'option `-f` permet de faire échouer la commande en cas de réponse HTTP correspondant à une erreur serveur ou client.

| Test                | Commande ou méthode                                      | Résultat attendu                |
| ------------------- | -------------------------------------------------------- | ------------------------------- |
| Résolution DNS      | `nslookup eval-dfs-p-tpl-20265-08.it-students.fr`        | Adresse IP du serveur retournée |
| Disponibilité HTTPS | `curl -f https://eval-dfs-p-tpl-20265-08.it-students.fr` | Requête réussie                 |
| Application Laravel | Requête HTTPS                                            | Application accessible          |
| Service Next.js     | Accès au dashboard                                       | Dashboard accessible            |
| Services système    | `systemctl status ...`                                   | Services actifs                 |

Un smoke test plus précis peut également vérifier le code HTTP retourné :

```bash
curl -s -o /dev/null \
  -w "%{http_code}" \
  -L https://eval-dfs-p-tpl-20265-08.it-students.fr
```

Le résultat attendu est :

```text
200
```

### 6.2 Preuve de déploiement réussi

La réussite du déploiement est vérifiée à partir de plusieurs éléments :

* le workflow GitHub Actions termine avec succès ;
* les contrôles préalables sont validés ;
* le commit déployé est identifiable ;
* les commandes de migration et de cache se terminent sans erreur ;
* les services nécessaires restent actifs ;
* le smoke test HTTPS retourne une réponse valide.

Le commit présent sur le serveur peut être vérifié avec :

```bash
git rev-parse HEAD
```

Cette valeur permet de comparer la version effectivement présente sur le serveur avec le commit ayant déclenché le pipeline.

Une capture de l'exécution GitHub Actions et de son résultat `Success` sera ajoutée comme preuve du déploiement.

Une capture ou sortie du smoke test peut également être conservée :

```text
HTTP 200
```

---

## 7. Conduite à tenir en cas d'échec

Le pipeline est conçu pour arrêter automatiquement la chaîne dès qu'une étape critique échoue.

### Échec avant le déploiement

Si les tests ou les contrôles préalables échouent :

```text
Tests
  │
  └── Échec
       │
       ▼
     STOP
```

Aucune modification n'est effectuée sur la production.

Les logs GitHub Actions sont analysés afin d'identifier l'origine du problème.

### Échec pendant le déploiement

En cas d'erreur pendant l'exécution du script :

* les logs sont consultés ;
* le service concerné est vérifié ;
* la configuration est contrôlée ;
* le commit déployé est identifié ;
* l'état de la base de données est vérifié avant toute opération corrective.

Les causes possibles comprennent notamment :

* dépendance manquante ;
* erreur Composer ou npm ;
* erreur de migration ;
* problème de permissions ;
* mauvaise configuration ;
* service indisponible ;
* erreur réseau.

### Échec du smoke test

Si le déploiement est terminé mais que le smoke test échoue, la version précédente peut être restaurée.

Le script conserve le commit précédemment déployé :

```bash
PREVIOUS_COMMIT=$(git rev-parse HEAD)
```

En cas d'échec, le serveur peut revenir à cette version :

```bash
git reset --hard "$PREVIOUS_COMMIT"
```

Les dépendances et caches nécessaires sont ensuite reconstruits avant de relancer le smoke test.

Le rollback doit être effectué avec prudence lorsqu'une migration de base de données a été exécutée. Une migration n'est pas automatiquement réversible et ne doit pas être annulée sans vérifier les conséquences sur les données.

La procédure générale est donc :

```text
Déploiement
     │
     ▼
Smoke test
     │
 ┌───┴────┐
 │        │
 OK      Échec
 │        │
 ▼        ▼
Succès   Diagnostic
          │
          ▼
       Rollback
          │
          ▼
      Smoke test
          │
       ┌──┴──┐
       │     │
      OK    KO
       │     │
       ▼     ▼
    Service  Intervention
    rétabli  manuelle
```

---

## 8. Scripts et fichiers de configuration

| Fichier                        | Rôle                                                                                             |
| ------------------------------ | ------------------------------------------------------------------------------------------------ |
| `.github/workflows/deploy.yml` | Définit le pipeline GitHub Actions, ses déclencheurs, ses contrôles et les étapes de déploiement |
| `scripts/deploy.sh`            | Regroupe les opérations exécutées sur le serveur de qualification ou de production               |
| `composer.json`                | Définit les dépendances PHP                                                                      |
| `composer.lock`                | Verrouille les versions des dépendances PHP                                                      |
| `package.json`                 | Définit les dépendances du dashboard Next.js                                                     |
| `package-lock.json`            | Verrouille les versions des dépendances JavaScript                                               |
| `.env.example`                 | Documente les variables d'environnement nécessaires sans contenir de secrets                     |
| `phpunit.xml`                  | Configure l'exécution des tests Laravel/PHPUnit                                                  |

Les secrets nécessaires au déploiement sont stockés dans les **GitHub Actions Secrets**.

Ils peuvent notamment comprendre :

| Secret            | Rôle                                      |
| ----------------- | ----------------------------------------- |
| `SSH_PRIVATE_KEY` | Clé privée utilisée pour la connexion SSH |
| `QUALIF_HOST`     | Adresse du serveur de qualification       |
| `PROD_HOST`       | Adresse du serveur de production          |
| `DEPLOY_USER`     | Utilisateur utilisé pour le déploiement   |

Les valeurs réelles des secrets ne sont jamais écrites dans le dépôt.

Les serveurs doivent disposer du dépôt de l'application et de l'environnement nécessaire à l'exécution du script.

Le compte utilisé pour le déploiement dispose uniquement des permissions nécessaires à l'exécution des opérations prévues.

---

## 9. Limites et évolutions

La chaîne actuelle privilégie une architecture simple et adaptée à la taille du projet.

Le déploiement repose sur un VPS unique pour l'environnement de production. Cette solution constitue un point unique de défaillance mais répond à la contrainte économique du projet.

Le smoke test actuel vérifie principalement que l'application répond correctement en HTTPS. Il pourrait être amélioré avec un endpoint de santé dédié, par exemple :

```text
/health
```

Cet endpoint pourrait vérifier plusieurs composants de l'application, notamment :

* disponibilité de Laravel ;
* connexion à MySQL ;
* disponibilité de Redis ;
* disponibilité de MongoDB ;
* état des services nécessaires.

Une évolution supplémentaire serait de mettre en place une stratégie de déploiement plus avancée, par exemple avec :

* validation manuelle obligatoire avant production ;
* versionnement des releases ;
* conservation de plusieurs versions précédentes ;
* déploiement sans interruption de service ;
* surveillance automatique après déploiement ;
* alertes en cas d'échec.

La supervision et la journalisation plus poussées sont traitées dans le livrable consacré au maintien en production.
