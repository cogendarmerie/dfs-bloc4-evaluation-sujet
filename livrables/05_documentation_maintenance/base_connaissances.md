# Base de connaissances — Note de passation

## 1. Présentation de l'application

**OpsTrack Field Service** est une application de gestion d'interventions permettant de centraliser le suivi des tickets, des techniciens et des interventions.

L'application est composée de plusieurs éléments :

* un backend Laravel assurant la logique métier et l'exposition de l'API REST ;
* une base de données MySQL pour les données métier ;
* MongoDB pour la journalisation et les événements techniques ;
* Redis pour les données temporaires et le cache ;
* un frontend Next.js `dispatch-dashboard` consommant l'API Laravel ;
* des intégrations avec des services externes ;
* un endpoint webhook permettant de recevoir des événements externes.

L'application est déployée dans un environnement de production sur une machine unique pour répondre aux contraintes économiques de l'épreuve.

---

# 2. Architecture technique

## 2.1 Composants principaux

| Composant                              | Technologie                  | Rôle                                                    |
| -------------------------------------- | ---------------------------- | ------------------------------------------------------- |
| Backend                                | Laravel 12 / PHP             | Logique métier, interface web et API REST               |
| Frontend                               | Next.js                      | Interface `dispatch-dashboard`                          |
| Base relationnelle                     | MySQL                        | Stockage des données métier                             |
| Base NoSQL                             | MongoDB                      | Stockage des événements et journaux techniques          |
| Cache                                  | Redis                        | Cache et données temporaires                            |
| API                                    | Laravel REST API             | Communication avec le frontend et les systèmes externes |
| Webhook                                | Laravel / `public/hooks.php` | Réception d'événements externes                         |
| Serveur web                            | Apache                       | Exposition HTTP/HTTPS de l'application                  |
| Gestionnaire de dépendances PHP        | Composer                     | Gestion des dépendances Laravel                         |
| Gestionnaire de dépendances JavaScript | npm                          | Gestion des dépendances Next.js                         |

---

## 2.2 Schéma d'architecture

```text
                         INTERNET
                            |
                            | HTTPS
                            v
                    +----------------+
                    |     Apache     |
                    +-------+--------+
                            |
              +-------------+-------------+
              |                           |
              v                           v
     +------------------+        +------------------+
     | Backend Laravel  |        |    Next.js       |
     | PHP / API REST   |<-------| dispatch-dashboard|
     +--------+---------+        +------------------+
              |
       +------+------+----------------+
       |      |      |                |
       v      v      v                v
    +------+ +------+ +-------+  +-----------+
    |MySQL | |MongoDB| | Redis |  | Services  |
    |Métier| |Logs   | |Cache  |  | externes  |
    +------+ +------+ +-------+  +-----------+
                                        ^
                                        |
                                   API externe
```

Le frontend Next.js ne communique pas directement avec les bases de données. Les échanges avec les données métier passent par le backend Laravel et son API.

---

# 3. Points d'attention connus

## 3.1 Webhook

Le fichier `public/hooks.php` constitue le point d'entrée du webhook.

Une anomalie a été identifiée lors de l'épreuve : la requête HTTP n'était pas transmise au kernel Laravel avant l'appel du contrôleur.

Le code initial effectuait directement :

```php
$request = Request::capture();
$response = $app->make(WebhookController::class)->handle($request);
```

Cela provoquait notamment l'erreur :

```text
ReflectionException: Class "config" does not exist
```

Le correctif consiste à faire passer la requête par le kernel Laravel :

```php
$request = Request::capture();
$kernel->handle($request);

$response = $app->make(WebhookController::class)->handle($request);
```

Après correction, le webhook a correctement retourné :

```json
{
    "message": "Webhook processed.",
    "intervention_id": 8
}
```

Cette correction doit être conservée lors des prochains déploiements.

---

## 3.2 Dépendances Composer

Un audit de sécurité initial avec :

```bash
composer audit
```

avait détecté :

```text
39 security vulnerability advisories affecting 12 packages
```

Une vulnérabilité concernant `league/commonmark` a été corrigée pendant l'épreuve en mettant à jour :

```text
league/commonmark
2.8.1 -> 2.10.1
```

La mise à jour a également entraîné la mise à jour de plusieurs dépendances indirectes.

Un nouvel audit a ensuite été effectué.

Résultat :

```text
28 security vulnerability advisories affecting 11 packages
```

`league/commonmark` n'apparaît plus dans l'audit final.

Les vulnérabilités restantes constituent un point de maintenance à traiter ultérieurement.

---

## 3.3 Tests automatisés

Les tests peuvent être exécutés avec :

```bash
php artisan test
```

Résultat obtenu après les corrections :

```text
Tests: 4 passed (6 assertions)
```

Les tests couvrent notamment :

* le fonctionnement général de l'application ;
* l'accès à l'application ;
* l'endpoint de santé ;
* l'authentification de l'API des tickets.

---

## 3.4 Infrastructure

La production repose actuellement sur une machine unique.

Cette architecture répond à la contrainte économique de l'épreuve mais constitue un point de vigilance :

* panne du serveur pouvant entraîner une indisponibilité globale ;
* ressources CPU, mémoire et stockage limitées ;
* bases de données et application présentes sur le même serveur ;
* absence de haute disponibilité.

Une évolution future pourrait consister à séparer progressivement les composants les plus critiques.

---

# 4. Procédures opérationnelles

## 4.1 Déploiement

Le déploiement est automatisé via un pipeline CI/CD utilisant GitHub Actions.

Le principe général est :

```text
Commit / déclenchement
        |
        v
Tests et contrôles
        |
        v
Déploiement qualification
        |
        v
Smoke test
        |
        v
Déploiement production
        |
        v
Smoke test production
```

Avant un déploiement manuel, vérifier notamment :

```bash
git status
php artisan test
composer audit
```

Pour vérifier les routes API :

```bash
php artisan route:list --path=api
```

Après déploiement, vérifier l'endpoint de santé :

```bash
curl -i https://eval-dfs-p-tpl-20265-08.it-students.fr/api/health
```

Une réponse HTTP `200 OK` indique que l'endpoint est disponible.

---

## 4.2 Sauvegarde et restauration

Les données critiques à sauvegarder sont principalement :

* la base MySQL ;
* les données MongoDB ;
* les fichiers applicatifs nécessaires à la restauration ;
* les fichiers de configuration et secrets, selon la politique de sauvegarde retenue.

### Sauvegarde MySQL

Exemple :

```bash
mysqldump -u <USER> -p <DATABASE> > backup.sql
```

### Sauvegarde MongoDB

Exemple :

```bash
mongodump --db <DATABASE> --out ./backup
```

### Restauration MySQL

```bash
mysql -u <USER> -p <DATABASE> < backup.sql
```

### Restauration MongoDB

```bash
mongorestore --db <DATABASE> ./backup/<DATABASE>
```

Les sauvegardes doivent être stockées sur un emplacement distinct du serveur de production afin qu'une panne du serveur ne détruise pas simultanément les données et leurs sauvegardes.

---

## 4.3 Supervision et alertes

Les premières vérifications à effectuer en cas d'incident sont :

### État des services

```bash
systemctl --failed
```

### Journaux système

```bash
journalctl -p err -b --no-pager
```

### Ports ouverts

```bash
ss -tulpn
```

### Logs Laravel

```bash
tail -n 100 /var/www/html/storage/logs/laravel.log
```

### État du firewall

```bash
ufw status verbose
```

### État de Fail2Ban

```bash
fail2ban-client status
```

### Tests applicatifs

```bash
php artisan test
```

### Audit des dépendances

```bash
composer audit
```

La première sonde applicative à utiliser est :

```text
GET /api/health
```

Elle permet de déterminer rapidement si l'application répond.

---

## 4.4 Accès et secrets

Les accès d'administration au serveur doivent être réalisés via SSH.

Les bonnes pratiques à respecter sont :

* utiliser l'authentification par clé SSH ;
* désactiver l'accès SSH direct par mot de passe lorsque possible ;
* limiter l'accès SSH aux adresses IP autorisées ;
* ne pas utiliser directement le compte `root` pour les opérations courantes ;
* utiliser des comptes disposant uniquement des droits nécessaires.

Les secrets applicatifs sont stockés dans l'environnement de production et ne doivent pas être versionnés.

Le fichier `.env` ne doit notamment pas être ajouté au dépôt Git.

En cas de compromission d'un secret, celui-ci doit être immédiatement révoqué et régénéré.

---

# 5. Bugs et failles corrigés pendant l'épreuve

## 5.1 Bug du webhook

### Symptôme

Le webhook provoquait une erreur lors de son exécution :

```text
ReflectionException: Class "config" does not exist
```

### Cause

La requête HTTP n'était pas passée par le kernel Laravel avant l'appel du `WebhookController`.

### Correction

Ajout de :

```php
$kernel->handle($request);
```

avant l'appel au contrôleur.

### Vérification

Le webhook retourne désormais :

```json
{
    "message": "Webhook processed.",
    "intervention_id": 8
}
```

---

## 5.2 Vulnérabilité de dépendance

### Détection

Un audit avec :

```bash
composer audit
```

a identifié plusieurs dépendances vulnérables.

`league/commonmark` en version `2.8.1` était notamment concerné par plusieurs vulnérabilités.

### Correction

Mise à jour vers :

```text
league/commonmark 2.10.1
```

avec :

```bash
composer update league/commonmark --with-dependencies
```

### Vérification

Les tests ont été exécutés :

```bash
php artisan test
```

Résultat :

```text
4 tests passed
6 assertions
```

Un nouvel audit Composer a également confirmé que `league/commonmark` n'était plus présent dans la liste des vulnérabilités.

---

# 6. Améliorations recommandées

## 6.1 Traitement des vulnérabilités restantes

L'audit final présente encore plusieurs vulnérabilités dans les dépendances PHP.

Une analyse complémentaire devra être menée afin de :

1. identifier les versions corrigées ;
2. vérifier la compatibilité avec Laravel 12 ;
3. mettre à jour progressivement les dépendances ;
4. exécuter les tests après chaque évolution ;
5. effectuer un nouvel `composer audit`.

Les mises à jour importantes doivent être réalisées en qualification avant toute mise en production.

---

## 6.2 Documentation Swagger / OpenAPI

La mise en place de Swagger/OpenAPI serait pertinente pour l'API.

Elle permettrait notamment de :

* documenter automatiquement les endpoints ;
* décrire les paramètres ;
* documenter les requêtes et réponses JSON ;
* documenter l'authentification ;
* tester les endpoints depuis une interface interactive.

Cette fonctionnalité n'a pas pu être mise en place pendant l'épreuve par manque de temps.

La documentation API actuelle a donc été réalisée manuellement à partir des routes et contrôleurs de l'application.

---

## 6.3 Supervision

La supervision pourrait être renforcée avec :

* une sonde applicative plus complète ;
* des alertes automatiques ;
* une surveillance CPU, mémoire et disque ;
* une surveillance des processus applicatifs ;
* une centralisation des logs ;
* un historique des incidents.

---

## 6.4 Sauvegardes

Une politique de sauvegarde formalisée devrait être mise en place avec :

* fréquence définie ;
* durée de rétention ;
* stockage externe ;
* chiffrement ;
* vérification automatique des sauvegardes ;
* tests réguliers de restauration.

Une sauvegarde qui n'a jamais été restaurée expérimentalement ne doit pas être considérée comme totalement fiable.

---

## 6.5 Haute disponibilité

La machine unique constitue actuellement un point de défaillance unique.

À moyen terme, l'architecture pourrait évoluer vers :

```text
                    Load Balancer
                         |
              +----------+----------+
              |                     |
          Serveur APP 1        Serveur APP 2
              |                     |
              +----------+----------+
                         |
                  Base de données
```

Cette évolution dépendrait cependant des besoins réels de disponibilité et du budget disponible.

---

# 7. Contacts et ressources

## Environnements

### Qualification

```text
https://eval-dfs-q-tpl-20265-08.it-students.fr
```

### Production

```text
https://eval-dfs-p-tpl-20265-08.it-students.fr
```

### Endpoint de santé

```text
https://eval-dfs-p-tpl-20265-08.it-students.fr/api/health
```

## Commandes utiles

### Routes

```bash
php artisan route:list --path=api
```

### Tests

```bash
php artisan test
```

### Audit des dépendances

```bash
composer audit
```

### Logs Laravel

```bash
tail -n 100 /var/www/html/storage/logs/laravel.log
```

### Logs système

```bash
journalctl -p err -b --no-pager
```

### Services en erreur

```bash
systemctl --failed
```

### Ports en écoute

```bash
ss -tulpn
```

### Firewall

```bash
ufw status verbose
```

### Fail2Ban

```bash
fail2ban-client status
```

## Documentation technique

Les ressources de référence à consulter lors d'une intervention sont :

* documentation Laravel ;
* documentation PHP ;
* documentation Composer ;
* documentation MySQL ;
* documentation MongoDB ;
* documentation Redis ;
* documentation Next.js ;
* documentation Apache ;
* documentation Git et GitHub.

---

# 8. Checklist de prise en charge d'un incident

En cas d'incident en production, suivre l'ordre suivant :

1. Vérifier la disponibilité :

```bash
curl -i https://eval-dfs-p-tpl-20265-08.it-students.fr/api/health
```

2. Vérifier les services :

```bash
systemctl --failed
```

3. Vérifier les journaux système :

```bash
journalctl -p err -b --no-pager
```

4. Vérifier les logs Laravel :

```bash
tail -n 100 /var/www/html/storage/logs/laravel.log
```

5. Vérifier les ports et services :

```bash
ss -tulpn
```

6. Vérifier le firewall :

```bash
ufw status verbose
```

7. Vérifier Fail2Ban :

```bash
fail2ban-client status
```

8. Vérifier les tests applicatifs :

```bash
php artisan test
```

9. Si le problème concerne une dépendance :

```bash
composer audit
```

10. Documenter le diagnostic, la cause, la correction et le résultat dans le journal de maintenance.

---

# 9. Règles générales de maintenance

Toute modification de production doit être :

* identifiée ;
* testée en qualification ;
* documentée ;
* déployée via le processus de déploiement prévu ;
* vérifiée après déploiement ;
* réversible lorsque cela est techniquement possible.

Les modifications importantes doivent être accompagnées d'une mise à jour du `CHANGELOG.md`.

Les corrections de sécurité doivent être reportées dans `SECURITY.md`.

La documentation technique et la présente base de connaissances doivent être mises à jour lorsque l'architecture ou les procédures d'exploitation évoluent.
