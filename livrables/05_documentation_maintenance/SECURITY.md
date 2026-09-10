# Journal de sécurité

Ce document recense les principales vulnérabilités et observations de sécurité identifiées pendant l'épreuve, ainsi que les mesures correctives appliquées ou recommandées.

---

## Faille 1 — Vulnérabilités de `league/commonmark`

| Élément                     | Information                                                                                                                                                           |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Date de détection           | 10 septembre 2026                                                                                                                                                     |
| Composant concerné          | `league/commonmark`                                                                                                                                                   |
| Version initiale            | `2.8.1`                                                                                                                                                               |
| Version corrigée            | `2.10.1`                                                                                                                                                              |
| Description de la faille    | Plusieurs vulnérabilités de sécurité affectaient la version utilisée, notamment des risques de XSS et de déni de service (DoS).                                       |
| Sévérité estimée            | Haute                                                                                                                                                                 |
| Impact potentiel            | Selon la vulnérabilité exploitée, un attaquant pourrait provoquer un déni de service ou exploiter un comportement de traitement de contenu potentiellement dangereux. |
| Mesure corrective appliquée | Mise à jour de `league/commonmark` vers la version `2.10.1` avec ses dépendances nécessaires.                                                                         |
| Statut                      | **Corrigée**                                                                                                                                                          |
| Preuve de correction        | `composer show league/commonmark` confirme la version `2.10.1`. Un nouvel `composer audit` ne signale plus `league/commonmark`.                                       |

### Diagnostic

Un premier audit des dépendances a été réalisé avec :

```bash
composer audit
```

Le résultat initial indiquait :

```text
Found 39 security vulnerability advisories affecting 12 packages.
```

`league/commonmark` en version `2.8.1` apparaissait parmi les composants concernés par plusieurs avis de sécurité.

Une simulation de mise à jour a d'abord été réalisée afin de vérifier les conséquences de la modification :

```bash
composer update league/commonmark --with-dependencies --dry-run
```

La simulation a indiqué que la mise à jour pouvait être effectuée avec un nombre limité de modifications de dépendances.

### Correctif

La mise à jour a ensuite été appliquée avec :

```bash
composer update league/commonmark --with-dependencies
```

La version a été mise à jour :

```text
league/commonmark 2.8.1 -> 2.10.1
```

Plusieurs dépendances indirectes ont également été mises à jour.

### Vérification

La version installée a été contrôlée avec :

```bash
composer show league/commonmark
```

Résultat :

```text
versions : * 2.10.1
```

Les tests automatisés ont ensuite été exécutés :

```bash
php artisan test
```

Résultat :

```text
Tests: 4 passed (6 assertions)
```

Enfin, un nouvel audit a été réalisé :

```bash
composer audit
```

Le nombre d'avis est passé de :

```text
39 advisories / 12 packages
```

à :

```text
28 advisories / 11 packages
```

`league/commonmark` n'apparaît plus dans l'audit final.

---

# Faille 2 — Vulnérabilités restantes des dépendances

| Élément                     | Information                                                                                                      |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Date de détection           | 10 septembre 2026                                                                                                |
| Composant concerné          | Plusieurs dépendances Composer                                                                                   |
| Description de la faille    | L'audit Composer identifie encore plusieurs vulnérabilités dans les dépendances installées.                      |
| Sévérité estimée            | De faible à haute selon la vulnérabilité                                                                         |
| Impact potentiel            | L'impact dépend de la dépendance et de la fonctionnalité concernée. Certaines vulnérabilités sont classées HIGH. |
| Mesure corrective appliquée | Analyse et correction ciblée de `league/commonmark`.                                                             |
| Statut                      | **À traiter**                                                                                                    |
| Preuve                      | Résultat final de `composer audit`.                                                                              |

Après la correction de `league/commonmark`, l'audit de sécurité final indique encore :

```text
28 security vulnerability advisories affecting 11 packages.
```

Les principaux composants encore concernés sont notamment :

* `guzzlehttp/guzzle` ;
* `guzzlehttp/psr7` ;
* `laravel/framework` ;
* `mongodb/mongodb` ;
* `symfony/http-foundation` ;
* `symfony/http-kernel` ;
* `symfony/mailer` ;
* `symfony/mime` ;
* `symfony/polyfill-intl-idn` ;
* `symfony/routing` ;
* `symfony/yaml`.

Certaines vulnérabilités sont classées **HIGH**, notamment dans Guzzle, Laravel, MongoDB et Symfony.

Ces dépendances n'ont pas été mises à jour de manière globale pendant l'épreuve afin de limiter le risque de régression et de préserver le temps disponible pour les autres opérations de maintenance.

### Action recommandée

Une analyse individuelle de chaque dépendance doit être réalisée afin de :

1. identifier les versions corrigées ;
2. vérifier la compatibilité avec Laravel 12 et les autres dépendances ;
3. effectuer les mises à jour progressivement ;
4. exécuter les tests après chaque modification ;
5. réaliser un nouvel `composer audit` ;
6. déployer les corrections en qualification avant la production.

Cette action constitue une priorité de maintenance à la suite de l'épreuve.

---

# 3. Sécurisation des accès SSH

Une analyse des journaux système a permis d'identifier plusieurs événements SSH inhabituels :

```text
Sep 10 04:19:39 ... sshd: error: kex_exchange_identification: read: Connection reset by peer
Sep 10 04:19:50 ... sshd: fatal: userauth_pubkey: parse publickey packet: incomplete message [preauth]
Sep 10 07:08:19 ... sshd: error: maximum authentication attempts exceeded for root from 147.224.162.134 ... [preauth]
Sep 10 13:17:31 ... sshd: error: kex_exchange_identification: read: Connection reset by peer
Sep 10 13:18:55 ... sshd: error: kex_exchange_identification: read: Connection reset by peer
```

Ces événements montrent notamment des tentatives de connexion SSH et une tentative ciblant le compte `root`.

Ils sont considérés comme des **événements de sécurité à surveiller**, sans permettre à eux seuls de conclure à une compromission du serveur.

### Mesures de protection

Le serveur utilise les mécanismes suivants :

* authentification SSH par clé ;
* restriction des adresses IP autorisées pour SSH ;
* désactivation de l'utilisation courante du compte `root` ;
* Fail2Ban pour détecter les tentatives répétées d'authentification ;
* firewall limitant les ports exposés.

L'état de Fail2Ban peut être vérifié avec :

```bash
fail2ban-client status
```

Les règles de firewall peuvent être contrôlées avec :

```bash
ufw status verbose
```

---

# 4. Gestion des secrets

Les secrets applicatifs ne doivent pas être versionnés dans Git.

Les informations sensibles telles que :

* tokens API ;
* mots de passe de bases de données ;
* clés privées ;
* secrets Laravel ;
* identifiants de services externes ;

doivent être conservées dans l'environnement de production ou dans un gestionnaire de secrets adapté.

Le fichier `.env` ne doit pas être commité dans le dépôt.

En cas de fuite d'un secret, celui-ci doit être considéré comme compromis et être immédiatement révoqué puis régénéré.

---

# 5. Vérifications de sécurité réalisées

Plusieurs contrôles ont été effectués pendant l'épreuve.

### Audit des dépendances

```bash
composer audit
```

### Tests applicatifs

```bash
php artisan test
```

Résultat :

```text
Tests: 4 passed (6 assertions)
```

### Vérification des routes API

```bash
php artisan route:list --path=api
```

### Vérification des ports ouverts

```bash
ss -tulpn
```

### Vérification du firewall

```bash
ufw status verbose
```

### Vérification de Fail2Ban

```bash
fail2ban-client status
```

### Consultation des journaux système

```bash
journalctl -p err -b --no-pager
```

---

# 6. Bilan de sécurité

Les contrôles réalisés pendant l'épreuve ont permis d'identifier et de traiter une vulnérabilité de dépendance importante concernant `league/commonmark`.

Le correctif a été validé par :

* la vérification de la version installée ;
* l'exécution des tests automatisés ;
* un nouvel audit Composer.

Le système présente toutefois encore des vulnérabilités dans plusieurs dépendances. Celles-ci sont documentées et constituent un **plan d'action de maintenance prioritaire**.

Les événements SSH observés sont également pris en compte dans la supervision de l'infrastructure afin de détecter d'éventuelles tentatives d'accès non autorisées.

La sécurité de l'application doit être considérée comme un processus continu : les dépendances, journaux, accès, services exposés et configurations doivent être régulièrement contrôlés.
