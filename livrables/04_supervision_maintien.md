# Supervision, journalisation, sauvegarde et maintenance corrective

> Competence evaluee : `C32` — Mettre en oeuvre un systeme de supervision pour detecter, diagnostiquer et corriger bugs, incidents et failles.

## 1. Journalisation

### 1.1 Services journalises

La journalisation permet de suivre l'etat de l'application, des services systeme et des acces au serveur. Elle constitue un element essentiel pour diagnostiquer les incidents et identifier d'eventuelles tentatives d'acces non autorisees.

| Service       | Emplacement des journaux                 | Niveau de detail                                                   |
| ------------- | ---------------------------------------- | ------------------------------------------------------------------ |
| Laravel       | `/var/www/html/storage/logs/laravel.log` | Erreurs applicatives, exceptions et informations de fonctionnement |
| Systeme Linux | `journald`                               | Evenements systeme et erreurs des services                         |
| SSH / OpenSSH | `journald`                               | Connexions, erreurs d'authentification et tentatives d'acces       |
| Serveur web   | Journaux du serveur web                  | Requetes HTTP et erreurs serveur                                   |
| Fail2Ban      | Journaux systeme / journald              | Tentatives detectees et actions de bannissement                    |

Les journaux applicatifs sont notamment utilises pour analyser les erreurs Laravel. Les journaux systeme permettent quant a eux de distinguer un probleme applicatif d'un probleme lie a l'infrastructure.

### 1.2 Configuration de la journalisation

Les journaux systeme sont exploites avec `journald`.

Une analyse des erreurs du demarrage courant a ete realisee avec :

```bash
journalctl -p err -b --no-pager
```

Cette commande a permis d'identifier plusieurs evenements lies au service SSH :

```text
Sep 10 04:19:39 ip-172-31-41-177 sshd[5421]: error: kex_exchange_identification: read: Connection reset by peer
Sep 10 04:19:50 ip-172-31-41-177 sshd[5422]: fatal: userauth_pubkey: parse publickey packet: incomplete message [preauth]
Sep 10 07:08:19 ip-172-31-41-177 sshd[52047]: error: maximum authentication attempts exceeded for root from 147.224.162.134 port 40252 ssh2 [preauth]
Sep 10 13:17:31 ip-172-31-41-177 sshd[53825]: error: kex_exchange_identification: read: Connection reset by peer
Sep 10 13:18:55 ip-172-31-41-177 sshd[53826]: error: kex_exchange_identification: read: Connection reset by peer
```

Ces traces mettent notamment en evidence :

* plusieurs connexions SSH interrompues pendant la phase d'etablissement de la connexion ;
* une tentative d'authentification avec un paquet de cle publique incomplet ;
* une tentative de connexion visant directement le compte `root` ;
* une tentative provenant de l'adresse IP `147.224.162.134` ;
* le depassement du nombre maximal de tentatives d'authentification.

L'evenement suivant est particulierement significatif :

```text
maximum authentication attempts exceeded for root from 147.224.162.134
```

Cette information constitue un indicateur d'une tentative d'acces non autorise ou d'une activite automatisee de type brute-force.

Les journaux Laravel peuvent etre consultes avec :

```bash
tail -n 100 /var/www/html/storage/logs/laravel.log
```

Les journaux systeme peuvent egalement etre consultes avec `journalctl` afin de rechercher des erreurs liees aux services :

```bash
journalctl -p err -b --no-pager
```

La combinaison des journaux applicatifs et systeme permet ainsi de disposer d'informations exploitables pour le diagnostic.

---

## 2. Outils et configurations d'audit

Plusieurs outils sont utilises pour controler l'etat de l'application, des services et des dependances.

### Audit des dependances

L'outil Composer permet d'identifier les vulnerabilites connues des dependances PHP :

```bash
composer audit
```

Cet audit a ete utilise lors de la maintenance corrective afin d'identifier les dependances vulnerables.

La version d'une dependance peut ensuite etre controlee avec :

```bash
composer show league/commonmark
```

### Tests applicatifs

La suite de tests Laravel est executee avec :

```bash
php artisan test
```

Elle permet de verifier qu'une modification du code ou des dependances n'introduit pas de regression dans les fonctionnalites couvertes.

### Etat des services

L'etat des services systeme peut etre controle avec :

```bash
systemctl --failed
```

Les ports ouverts et services en ecoute peuvent etre identifies avec :

```bash
ss -tulpn
```

Ces commandes permettent de verifier l'etat general du serveur et d'identifier un eventuel service inattendu ou indisponible.

### Audit du pare-feu

L'etat du pare-feu peut etre verifie avec :

```bash
ufw status verbose
```

Cette verification permet notamment de controler les ports exposes sur le serveur.

### Audit Fail2Ban

Fail2Ban permet de controler les protections actives avec :

```bash
fail2ban-client status
```

Le service peut egalement etre interroge pour obtenir l'etat d'une protection particuliere, notamment celle concernant SSH.

---

## 3. Supervision et alertes

### 3.1 Sondes mises en place

La supervision repose sur des controles permettant de verifier la disponibilite de l'application, l'etat des services et la securite du serveur.

| Sonde              | Cible               | Seuil ou condition                        | Action en cas d'alerte                            |
| ------------------ | ------------------- | ----------------------------------------- | ------------------------------------------------- |
| Disponibilite HTTP | Application Laravel | Reponse HTTP en erreur ou indisponibilite | Analyse du serveur web et des logs Laravel        |
| Health check       | Endpoint `/health`  | Reponse differente de `200 OK`            | Diagnostic de l'application et de ses dependances |
| Etat des services  | Services systeme    | Service en echec ou arrete                | Analyse avec `systemctl` et `journalctl`          |
| Espace disque      | Systeme de fichiers | Espace disponible insuffisant             | Analyse et nettoyage des fichiers/journaux        |
| SSH                | Service OpenSSH     | Tentatives d'authentification anormales   | Analyse des logs et protection Fail2Ban           |
| Fail2Ban           | Protection SSH      | Tentatives repetitives detectees          | Bannissement selon la configuration               |
| Dependances PHP    | Composer            | Presence d'advisories de securite         | Analyse et mise a jour des dependances            |

L'application dispose notamment d'un endpoint `/health`, permettant de verifier rapidement son bon fonctionnement.

Ce endpoint est egalement couvert par la suite de tests Laravel.

### 3.2 Mecanisme d'alerte

La supervision suit une logique de detection, diagnostic puis correction.

Lorsqu'une anomalie est detectee :

1. l'anomalie est confirmee ;
2. le composant concerne est identifie ;
3. les journaux correspondants sont consultes ;
4. le probleme est reproduit lorsque cela est possible ;
5. la cause racine est recherchee ;
6. un correctif est applique ;
7. les tests necessaires sont executes ;
8. le service est controle apres correction.

Pour les problemes de securite, l'audit `composer audit` permet egalement d'identifier les dependances vulnerables.

---

## 4. Strategie de sauvegarde et restauration

### 4.1 Elements sauvegardes

Les donnees critiques de l'application doivent etre sauvegardees afin de permettre une restauration en cas de panne, de corruption ou de perte de donnees.

| Element                               | Methode                                        | Frequence                                      | Retention                    |
| ------------------------------------- | ---------------------------------------------- | ---------------------------------------------- | ---------------------------- |
| Base MySQL                            | `mysqldump`                                    | Selon la politique de sauvegarde de production | Selon politique de retention |
| Base MongoDB                          | `mongodump`                                    | Selon la politique de sauvegarde de production | Selon politique de retention |
| Donnees persistantes de l'application | Sauvegarde des fichiers necessaires            | Selon besoin                                   | Selon politique de retention |
| Code source                           | Git                                            | A chaque modification                          | Historique Git               |
| Configuration de deploiement          | Git / sauvegarde des fichiers de configuration | A chaque modification                          | Historique Git               |

Les bases MySQL et MongoDB doivent etre sauvegardees separement du code source.

Le code source et les fichiers de configuration versionnes dans Git permettent quant a eux de reconstruire l'application, tandis que les sauvegardes des bases permettent de conserver les donnees produites par les utilisateurs et les traitements.

### 4.2 Procedure de restauration

En cas de perte ou de corruption de donnees, la procedure generale est la suivante :

1. Identifier le composant concerne et l'etendue de l'incident.
2. Isoler le composant si necessaire afin d'eviter d'aggraver l'incident.
3. Identifier la derniere sauvegarde valide.
4. Verifier l'integrite de la sauvegarde avant restauration.
5. Restaurer la base de donnees concernee.
6. Verifier la coherence des donnees restaurees.
7. Redemarrer les services necessaires.
8. Executer les tests applicatifs.
9. Executer le health check.
10. Verifier le fonctionnement de l'application.
11. Documenter l'incident et les actions realisees.

La procedure de restauration doit etre testee periodiquement afin de verifier que les sauvegardes sont effectivement exploitables.

---

## 5. Diagnostic et correction du bug technique

### 5.1 Symptome observe

Un dysfonctionnement a ete constate lors du traitement du webhook de l'application.

L'appel du webhook provoquait l'erreur suivante :

```text
Fatal error: Uncaught ReflectionException: Class "config" does not exist
```

La trace d'erreur indiquait notamment :

```text
WebhookController.php(20): config('services.webhoo...')
public/hooks.php(13): App\Http\Controllers\WebhookController->handle(...)
```

Le probleme intervenait donc lors de l'execution du `WebhookController`, appele depuis le point d'entree `public/hooks.php`.

### 5.2 Demarche de diagnostic

La trace d'erreur a d'abord permis d'identifier le chemin d'execution :

```text
public/hooks.php
       |
       v
WebhookController->handle()
       |
       v
config(...)
       |
       v
Container Laravel
       |
       v
ReflectionException
```

L'analyse du fichier `public/hooks.php` a ensuite permis d'identifier une difference dans l'initialisation du cycle HTTP Laravel.

La requete etait capturee :

```php
$request = Request::capture();
```

puis le contrôleur etait appele directement :

```php
$response = $app->make(WebhookController::class)->handle($request);
```

Le kernel HTTP Laravel n'etait pas appele entre ces deux etapes.

L'hypothese retenue a donc ete que l'application executait le contrôleur avant que le contexte HTTP Laravel soit correctement initialise.

### 5.3 Cause racine identifiee

La cause racine etait l'absence du traitement de la requete par le kernel HTTP Laravel avant l'appel au contrôleur.

Le fichier capturait correctement la requete mais ne l'envoyait pas au kernel :

```php
$request = Request::capture();
```

Il manquait l'appel :

```php
$kernel->handle($request);
```

Le contrôleur utilisait notamment la fonction `config()`. D'apres la trace d'erreur, cette fonction etait appelee alors que le contexte necessaire a sa resolution n'etait pas correctement initialise.

### 5.4 Correctif applique

Le traitement de la requete par le kernel Laravel a ete ajoute avant l'appel au contrôleur.

Le fichier corrige est devenu :

```php
<?php

use App\Http\Controllers\WebhookController;
use Illuminate\Http\Request;

require __DIR__.'/../vendor/autoload.php';

$app = require_once __DIR__.'/../bootstrap/app.php';
$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);

$request = Request::capture();
$kernel->handle($request);

$response = $app->make(WebhookController::class)->handle($request);
$response->send();

$kernel->terminate($request, $response);
```

Le cycle de traitement est maintenant :

```text
Requete HTTP
     |
     v
Request::capture()
     |
     v
Kernel Laravel
     |
     v
WebhookController
     |
     v
Response
     |
     v
send()
     |
     v
terminate()
```

### 5.5 Verification apres correction

Apres application du correctif, le webhook a ete execute avec succes.

La reponse obtenue est :

```json
{
  "message": "Webhook processed.",
  "intervention_id": 8
}
```

Cette reponse confirme que le traitement du webhook arrive maintenant jusqu'a son terme et qu'une intervention a bien ete traitee.

Une suite de tests Laravel a ensuite ete executee :

```bash
php artisan test
```

Resultat :

```text
PASS Tests\Unit\ExampleTest
PASS Tests\Feature\ExampleTest

Tests: 4 passed (6 assertions)
Duration: 0.27s
```

Les tests disponibles ont donc ete executes avec succes apres le correctif.

---

## 6. Diagnostic et correction de la faille de securite

### 6.1 Faille identifiee

Un audit des dependances PHP a ete realise avec :

```bash
composer audit
```

L'audit initial a signale :

```text
Found 39 security vulnerability advisories affecting 12 packages
```

Parmi les dependances concernees se trouvait `league/commonmark`.

La version effectivement installee a ete verifiee avec :

```bash
composer show league/commonmark
```

La version identifiee etait :

```text
league/commonmark 2.8.1
```

L'audit indiquait plusieurs vulnerabilites concernant cette version, dont des vulnerabilites de severite HIGH liees notamment a des risques de XSS et de deni de service.

### 6.2 Demarche de diagnostic

La demarche de diagnostic a ete realisee en plusieurs etapes.

Tout d'abord, l'audit global a ete execute :

```bash
composer audit
```

La dependance concernee a ensuite ete examinee :

```bash
composer show league/commonmark
```

La version `2.8.1` a ete comparee aux plages de versions affectees indiquees par Composer.

Avant d'appliquer la modification, une simulation a ete realisee :

```bash
composer update league/commonmark --with-dependencies --dry-run
```

La simulation a indique qu'une mise a jour etait possible :

```text
league/commonmark 2.8.1 => 2.10.1
```

La mise a jour concernait un nombre limite de dependances, ce qui permettait de reduire le risque de regression.

### 6.3 Evaluation du risque

Les vulnerabilites identifiees sur `league/commonmark` concernaient notamment :

* des risques de Cross-Site Scripting (XSS) ;
* des risques de deni de service (DoS) lors du traitement de contenus specifiquement construits.

Un risque XSS peut notamment devenir critique lorsqu'une application traite du contenu non fiable et le transforme en HTML avant de le restituer.

Un risque DoS peut entrainer une consommation excessive de ressources et affecter la disponibilite de l'application.

La presence d'advisories de severite HIGH justifiait donc la mise a jour de la dependance.

### 6.4 Mesure corrective appliquee

La mise a jour a ete executee avec :

```bash
composer update league/commonmark --with-dependencies
```

La version de `league/commonmark` a ete mise a jour :

```text
2.8.1 => 2.10.1
```

Des dependances indirectes ont egalement ete mises a jour :

```text
nette/schema                   v1.3.5 => v1.3.6
nette/utils                    v4.1.3 => v4.1.5
symfony/deprecation-contracts  v3.6.0 => v3.7.1
symfony/polyfill-php80         v1.33.0 => v1.37.0
```

Le fichier `composer.lock` a ete mis a jour afin de conserver les versions resultant de la resolution des dependances.

La version finale a ete controlee avec :

```bash
composer show league/commonmark
```

Resultat :

```text
versions : * 2.10.1
```

### 6.5 Verification apres correction

Une suite de tests a ete executee apres la mise a jour :

```bash
php artisan test
```

Resultat :

```text
Tests: 4 passed (6 assertions)
Duration: 0.27s
```

Aucune regression n'a ete detectee par les tests disponibles.

Un nouvel audit a ensuite ete realise :

```bash
composer audit
```

Le resultat final est passe de :

```text
39 security vulnerability advisories affecting 12 packages
```

a :

```text
28 security vulnerability advisories affecting 11 packages
```

`league/commonmark` n'apparait plus dans la liste des dependances vulnerables retournee par l'audit final.

La correction de cette dependance est donc verifiee par un nouvel audit de securite et par l'execution des tests applicatifs.

---

## 7. Autres observations

L'audit de securite final fait encore apparaitre plusieurs dependances necessitant une analyse ulterieure.

Les packages encore signales comprennent notamment :

* `guzzlehttp/guzzle` ;
* `guzzlehttp/psr7` ;
* `laravel/framework` ;
* `mongodb/mongodb` ;
* `symfony/http-foundation` ;
* `symfony/http-kernel` ;
* `symfony/mailer` ;
* `symfony/mime` ;
* `symfony/routing` ;
* `symfony/yaml` ;
* `symfony/polyfill-intl-idn`.

Ces vulnerabilites n'ont pas ete traitees dans le cadre de cette correction afin de limiter le perimetre des modifications et de ne pas introduire de changements de versions plus importants sans analyse de compatibilite.

Elles constituent des actions de maintenance complementaires. Chaque mise a jour devra etre precedee d'une verification des contraintes de dependances et suivie :

1. d'une mise a jour controlee ;
2. de l'execution de `php artisan test` ;
3. d'un controle fonctionnel de l'application ;
4. d'un nouvel `composer audit`.

L'analyse des journaux systeme a egalement mis en evidence plusieurs evenements SSH suspects, notamment une tentative d'authentification visant le compte `root` depuis l'adresse `147.224.162.134`.

L'evenement :

```text
maximum authentication attempts exceeded for root
```

constitue un indicateur de tentative d'acces non autorise ou d'activite automatisee.

La presence de Fail2Ban permet de completer les mesures de securite du serveur en detectant les comportements repetitifs et en appliquant des bannissements selon sa configuration.

Ces observations demontrent l'interet de combiner :

* journalisation applicative ;
* journalisation systeme ;
* supervision des services ;
* controle des dependances ;
* protection contre les tentatives d'authentification ;
* tests automatises ;
* verification fonctionnelle apres correction.

Le correctif du webhook et la mise a jour de `league/commonmark` ont tous deux ete suivis d'une verification afin de confirmer le retour a un fonctionnement nominal et de limiter le risque de regression.
