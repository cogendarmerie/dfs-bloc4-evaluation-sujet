# Changelog

Toutes les modifications notables apportées pendant l'épreuve sont documentées dans ce fichier.

Le format s'inspire de [Keep a Changelog](https://github.com/olivierlacan/keep-a-changelog).

## [Session du 10 septembre 2026]

### Ajouté

* Ajout de la gestion du cycle HTTP Laravel dans `public/hooks.php` avec l'appel au kernel avant l'exécution du `WebhookController`.
* Ajout de contrôles de validation de l'application avec `php artisan test`.
* Ajout d'un contrôle de sécurité des dépendances avec `composer audit`.
* Ajout et mise à jour de la documentation technique et de maintenance.
* Ajout de la documentation des endpoints REST disponibles dans `documentation_api.md`.
* Ajout des observations et procédures d'exploitation dans la base de connaissances.

### Modifié

* Modification de `public/hooks.php` afin de transmettre correctement la requête HTTP au kernel Laravel avant son traitement par le contrôleur du webhook.
* Mise à jour de `league/commonmark` de la version `2.8.1` vers `2.10.1`.
* Mise à jour de plusieurs dépendances indirectes nécessaires à la mise à jour de `league/commonmark`.
* Mise à jour du fichier `composer.lock` à la suite de la mise à jour des dépendances.
* Complément de la documentation de l'API avec les routes réellement exposées par Laravel :

  * `/api/health`
  * `/api/v1/tickets`
  * `/api/v1/technicians`
  * `/api/v1/external/weather`

### Corrigé

* Correction d'une erreur lors du traitement du webhook.

L'erreur initiale était notamment :

```text
ReflectionException: Class "config" does not exist
```

La cause identifiée était l'absence de traitement de la requête par le kernel Laravel avant l'appel du `WebhookController`.

Le traitement a été corrigé avec :

```php
$kernel->handle($request);
```

Le fonctionnement du webhook a ensuite été vérifié avec une réponse fonctionnelle :

```json
{
    "message": "Webhook processed.",
    "intervention_id": 8
}
```

* Vérification de non-régression après les modifications avec :

```bash
php artisan test
```

Résultat :

```text
Tests: 4 passed (6 assertions)
```

### Sécurité

* Réalisation d'un premier audit des dépendances avec :

```bash
composer audit
```

Résultat initial :

```text
39 security vulnerability advisories affecting 12 packages
```

* Identification de plusieurs vulnérabilités affectant notamment `league/commonmark` `2.8.1`.
* Analyse de la possibilité de mise à jour avec :

```bash
composer update league/commonmark --with-dependencies --dry-run
```

* Mise à jour effective de `league/commonmark` :

```text
2.8.1 -> 2.10.1
```

* Exécution des tests après la mise à jour :

```bash
php artisan test
```

Résultat :

```text
4 tests passed
6 assertions
```

* Réalisation d'un nouvel audit de sécurité :

```bash
composer audit
```

Résultat final :

```text
28 security vulnerability advisories affecting 11 packages
```

`league/commonmark` n'est plus signalé dans l'audit final.

Les vulnérabilités restantes dans les autres dépendances sont documentées dans `SECURITY.md` et constituent un plan d'action de maintenance à poursuivre.

* Analyse des journaux SSH ayant permis d'identifier plusieurs tentatives de connexion inhabituelles, notamment une tentative d'authentification visant le compte `root`.
* Vérification des mécanismes de protection de l'accès SSH, du firewall et de Fail2Ban.

### Documentation

* Création de `documentation_api.md` décrivant :

  * l'architecture de l'API ;
  * l'authentification ;
  * les endpoints disponibles ;
  * les paramètres ;
  * les exemples de requêtes ;
  * les codes HTTP ;
  * les mécanismes de journalisation.
* Création de `SECURITY.md` pour assurer le suivi des vulnérabilités identifiées et des corrections appliquées.
* Création de `base_connaissances.md` pour faciliter la reprise et la maintenance de l'application par un autre intervenant.
* Documentation des procédures de diagnostic, de supervision, de sauvegarde et de restauration.
* Identification de Swagger/OpenAPI comme amélioration recommandée pour fournir une documentation interactive de l'API.

### Améliorations à prévoir

Les évolutions suivantes ont été identifiées mais n'ont pas pu être réalisées pendant l'épreuve par manque de temps :

* mise en place d'une documentation interactive Swagger/OpenAPI ;
* traitement des vulnérabilités restantes détectées par `composer audit` ;
* renforcement et automatisation de la supervision ;
* formalisation et automatisation de la politique de sauvegarde ;
* amélioration de l'endpoint de santé ;
* augmentation de la couverture des tests automatisés ;
* évolution de l'architecture vers une infrastructure permettant une meilleure disponibilité.
