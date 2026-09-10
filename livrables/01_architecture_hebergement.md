# Architecture cible et choix de l'hebergement

> Competence evaluee : `C29` — Selectionner une plateforme d'hebergement adaptee aux exigences techniques, economiques, qualitatives et reglementaires.

## 1. Analyse des besoins techniques

OpsTrack Field Service est une application web composée de plusieurs services ayant des responsabilités distinctes.

Le cœur applicatif est développé avec **Laravel 12** et fournit l'interface web ainsi qu'une API REST. L'application utilise également un endpoint de webhook permettant de recevoir des événements provenant de systèmes externes.

Les données métier sont stockées dans **MySQL**, notamment les utilisateurs, tickets, interventions et commentaires.

**MongoDB** est utilisé pour les événements et journaux techniques nécessitant un stockage orienté document.

**Redis** est utilisé pour le cache, l'amélioration des performances et le stockage temporaire de certaines informations.

Un tableau de bord indépendant développé avec **Next.js** consomme l'API REST Laravel afin de fournir une interface dédiée au suivi et à la gestion des interventions.

L'application échange également avec des services externes via une API publique et reçoit des données par webhook.

L'architecture doit donc permettre de faire fonctionner simultanément ces différents composants sur une infrastructure unique, tout en assurant leur isolation logique et leur sécurité.

La contrainte économique impose initialement l'utilisation d'un **seul VPS**. Celui-ci doit disposer de suffisamment de ressources CPU, mémoire et stockage pour faire fonctionner Laravel, Next.js, MySQL, MongoDB et Redis simultanément.

Les performances attendues sont celles d'une application métier : temps de réponse raisonnable pour les opérations courantes, disponibilité des données métier et capacité à supporter une augmentation progressive du nombre d'utilisateurs et de tickets.

Le dimensionnement initial retenu est volontairement supérieur au strict minimum afin de conserver une marge de ressources pour les traitements Laravel, la base de données et les composants complémentaires.

## 2. Architecture cible proposee

### 2.1 Diagramme de deploiement

L'architecture cible initiale est organisée autour d'un VPS unique.

```text
                         INTERNET
                            |
                     DNS / Domaine
                            |
                         HTTPS
                            |
                    +---------------+
                    |     Nginx     |
                    | Reverse Proxy |
                    +-------+-------+
                            |
             +--------------+--------------+
             |                             |
             v                             v
      +-------------+              +---------------+
      |   Laravel   |              |    Next.js    |
      | PHP / API   |<-------------| Dashboard Web |
      +------+------+              +---------------+
             |
       +-----+----------+----------------+
       |                |                |
       v                v                v
   +-------+        +--------+       +---------+
   | MySQL |        | MongoDB|       |  Redis  |
   | Métier|        | Logs / |       | Cache / |
   |       |        | Events |       | temporaire
   +-------+        +--------+       +---------+
       |
       v
  Sauvegardes
  externalisées

        ^                         ^
        |                         |
 API publique                 Webhook externe
        |                         |
        +------------> Laravel / hooks.php
```

Les services de données ne sont pas directement exposés sur Internet. Les communications entre les composants s'effectuent sur le réseau interne du serveur.

Cette architecture correspond à la première étape de déploiement imposée par la contrainte économique. Elle pourra évoluer ultérieurement vers une architecture distribuée.

### 2.2 Description des composants

| Composant          | Service ou technologie | Dimensionnement                  | Justification                                             |
| ------------------ | ---------------------- | -------------------------------- | --------------------------------------------------------- |
| Reverse proxy      | Nginx                  | 1 instance                       | Point d'entrée HTTPS, routage vers Laravel et Next.js     |
| Application métier | Laravel 12 / PHP-FPM   | 1 instance                       | Cœur fonctionnel et API REST                              |
| Dashboard          | Next.js / Node.js      | 1 instance                       | Interface de supervision séparée consommant l'API Laravel |
| Base relationnelle | MySQL                  | 1 instance                       | Stockage des données métier transactionnelles             |
| Base NoSQL         | MongoDB                | 1 instance                       | Stockage des événements et journaux techniques            |
| Cache              | Redis                  | 1 instance                       | Cache et stockage temporaire                              |
| Système            | VPS Linux              | Ressources adaptées aux services | Hébergement de l'ensemble des composants                  |
| Sauvegarde         | Stockage externalisé   | Selon volumétrie                 | Protection contre la perte du VPS                         |

Le VPS constitue une infrastructure mutualisant les différents services, mais ceux-ci restent séparés au niveau applicatif et réseau.

## 3. Choix du fournisseur et des services

### 3.1 Fournisseur retenu

Le fournisseur retenu pour l'hébergement initial est **OVHcloud**, avec une offre VPS située en France ou dans l'Union européenne.

Le service principal retenu est un VPS Linux permettant d'administrer directement l'environnement et d'installer les différents composants nécessaires à OpsTrack.

Des services complémentaires sont prévus pour le nom de domaine et le stockage des sauvegardes.

### 3.2 Justification du choix

OVHcloud est retenu pour plusieurs raisons.

Tout d'abord, l'offre VPS permet de disposer d'une machine suffisamment flexible pour héberger l'ensemble des composants de l'application sur une infrastructure unique, conformément à la contrainte économique du projet.

Le VPS permet également d'augmenter les ressources CPU, mémoire ou stockage lorsque la charge augmente, ce qui constitue une première forme de scalabilité verticale.

Le choix d'un fournisseur français/européen facilite par ailleurs la maîtrise de la localisation des données et la prise en compte des exigences liées à la protection des données personnelles.

L'administration complète du serveur permet également de configurer précisément :

* le reverse proxy Nginx ;
* PHP et Laravel ;
* Node.js et Next.js ;
* MySQL ;
* MongoDB ;
* Redis ;
* le pare-feu ;
* les certificats TLS ;
* la supervision ;
* les sauvegardes.

Ce choix permet donc de conserver une infrastructure simple et économiquement maîtrisée tout en disposant d'une possibilité d'évolution.

## 4. Estimation des couts

L'estimation suivante correspond à l'infrastructure initiale et reste volontairement cohérente avec la contrainte économique du projet.

| Poste de depense             | Cout mensuel estime | Cout annuel estime |
| ---------------------------- | ------------------: | -----------------: |
| VPS Linux                    |                15 € |              180 € |
| Stockage des sauvegardes     |                 5 € |               60 € |
| Nom de domaine               |              1,25 € |               15 € |
| Certificat TLS Let's Encrypt |                 0 € |                0 € |
| **Total**                    |         **21,25 €** |          **255 €** |

Les montants sont des estimations permettant de comparer les coûts de l'architecture. Les tarifs réels peuvent varier selon l'offre et les ressources retenues.

L'utilisation de Let's Encrypt permet de mettre en place gratuitement les certificats TLS nécessaires à la sécurisation des communications HTTPS.

Le coût pourra augmenter lors d'une évolution vers une architecture distribuée nécessitant plusieurs serveurs ou des services managés supplémentaires.

## 5. Elasticite et evolutivite

L'architecture initiale privilégie la **scalabilité verticale**, car l'ensemble des composants est hébergé sur un seul VPS.

En cas d'augmentation de la charge, les ressources du VPS pourront dans un premier temps être augmentées : CPU, mémoire vive et espace de stockage.

Cette solution permet d'accompagner la croissance sans modifier immédiatement l'architecture applicative.

Si la croissance devient importante, une évolution vers une architecture distribuée pourra être réalisée :

1. séparation de Laravel et Next.js sur plusieurs instances ;
2. mise en place d'un reverse proxy ou load balancer ;
3. séparation des bases de données du serveur applicatif ;
4. utilisation éventuelle de services de bases de données managés ;
5. externalisation durable des sauvegardes ;
6. réplication des composants nécessitant une haute disponibilité.

Cette évolution permettra alors d'utiliser de la scalabilité horizontale lorsque cela sera nécessaire.

## 6. Disponibilite et continuite de service

L'infrastructure initiale repose sur un VPS unique. Cette solution constitue donc un **point unique de défaillance** et ne fournit pas le même niveau de disponibilité qu'une architecture redondante.

Pour limiter les conséquences d'une panne, les mesures suivantes sont prévues :

* redémarrage automatique des services ;
* supervision du serveur et des applications ;
* surveillance de l'espace disque, de la mémoire et du CPU ;
* journalisation des événements applicatifs et système ;
* sauvegardes régulières ;
* stockage des sauvegardes indépendamment du VPS de production ;
* procédure de restauration documentée.

L'objectif initial est de privilégier un niveau de disponibilité cohérent avec le budget et la criticité de l'application.

Une architecture à plusieurs instances pourra être mise en place ultérieurement afin de supprimer le point unique de défaillance et permettre un basculement automatique.

## 7. Securite et sauvegarde

Le serveur est protégé par un pare-feu afin de limiter les ports accessibles depuis Internet.

Seuls les services nécessaires à l'exploitation sont exposés publiquement, notamment :

* SSH pour l'administration ;
* HTTP pour la redirection vers HTTPS ;
* HTTPS pour l'accès à l'application.

Les ports de MySQL, MongoDB et Redis ne sont pas exposés directement sur Internet. Ces services doivent uniquement être accessibles depuis les composants internes qui en ont besoin.

Les communications avec les utilisateurs sont chiffrées avec TLS via HTTPS.

Les secrets applicatifs et identifiants de connexion aux bases de données sont stockés dans les variables d'environnement de production et ne sont pas versionnés dans Git.

Laravel est exécuté avec `APP_ENV=production` et `APP_DEBUG=false`.

Les comptes et services disposent uniquement des permissions nécessaires à leur fonctionnement, selon le principe du moindre privilège.

Les sauvegardes sont réalisées régulièrement et stockées sur un emplacement distinct du serveur de production afin qu'une compromission ou une panne du VPS ne provoque pas simultanément la perte des sauvegardes.

Une procédure de restauration doit être régulièrement vérifiée afin de s'assurer que les sauvegardes sont réellement exploitables.

## 8. Conformite et contraintes reglementaires

OpsTrack pouvant traiter des données relatives aux utilisateurs, aux tickets, aux interventions et aux commentaires, les principes du **RGPD** sont pris en compte dans la conception de l'infrastructure.

Les principales mesures retenues sont :

* collecter uniquement les données nécessaires au fonctionnement de l'application ;
* limiter l'accès aux données selon les rôles des utilisateurs ;
* protéger les échanges par HTTPS/TLS ;
* restreindre les accès aux bases de données ;
* protéger les sauvegardes ;
* limiter la conservation des journaux contenant potentiellement des données personnelles ;
* assurer une traçabilité des opérations sensibles ;
* limiter les droits d'administration de l'infrastructure ;
* prévoir les mécanismes permettant de répondre aux droits des personnes lorsque ceux-ci sont applicables.

Le choix d'un hébergeur français ou européen facilite également la maîtrise de la localisation des données et des éventuels transferts de données hors de l'Union européenne.

La journalisation doit par ailleurs être conçue de manière à assurer la traçabilité nécessaire à l'exploitation et à la sécurité, sans conserver inutilement des données personnelles.

L'architecture retenue cherche ainsi à respecter un équilibre entre sécurité, disponibilité, évolutivité, conformité réglementaire et maîtrise des coûts.
