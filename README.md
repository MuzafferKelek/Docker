# Docker
# Préambule

Cette page de wiki permettra d'apporter quelques informations sur quelques points concernant Docker et plus particulièrement sur les façons de faire votre Dockerfile. Je ferais la liste des images Docker de base sur lesquelles vous pourrez vous greffer pour vos projets. Ces images possèdent tout les outils de base que nous utilisons dans nos projets Symfony et devraient êtres adaptées dans 95% des cas.

# Images disponibles

Toutes mes images "génériques" sont hébergées sur Docker Hub afin d'être le plus possible dispos de partout. Vous trouverez la liste complète [ICI](https://hub.docker.com/repository/docker/soljian/php/tags?page=1&ordering=last_updated&name=gnlk).

Les images tagués `gnlk` sont basées sur Debian et celles taguées `gnlk-light` sont basées sur Alpine. Elles possèdent toutes les dépendances de bases à savoir :
- PHP
- NodeJS (LTS → 14.17.6)
- NPM (6.14.15)
- Yarn (1.22.4)
- Extensions PHP : gd, ldap, oci8, opcache, pdo_mysql, mysqli, pdo_sqlsrv, sqlsrv, redis, soap, sodium, zip
- Packages NPM globaux : node-gyp
- Composer (1.10.22)
- Oracle Instant Client (12.2.0.1.0)
- Connecteur SQLSRV ODBC (17)
- Scripts : wait-for-it, healthcheck

*(Référez vous aux [Dockerfiles](https://gitlab.genilink.com/interne/devops/images-docker/-/tree/master/php) si besoin de plus de détails)*

Au niveaux des versions de PHP disponibles :  

Debian : 7.1.33, 7.2, 7.3, 7.4, 8.0

Alpine : 7.2, 7.3, 7.4

# Créer son propre Dockerfile basé sur une image gnlk

Il est très simple de créer son propre Dockerfile quand on ne part pas de zéro. Pour ce faire il suffira de prendre une des images gnlk selon son besoin et d'ajouter ce qui est nécessaire sur son projet. Vous aurez quelques exemples disponibles ici et là mais le plus parlant est celui du [ws-auth-v2](https://gitlab.genilink.com/externe/webservices/ws-auth-v2/-/blob/master/Dockerfile). Vous pouvez sans soucis le copier coller et le customiser au besoin.

## Entête (FROM, LABEL, ENV, ARGS)

Dans chaque Dockerfile vous trouverez un `FROM`. Celui-ci indique de quel image votre Dockerfile hérite. Il est possible de partir de zéro en prenant une image d'OS comme `debian:buster` ou `centos:7` mais je vous encourage à partir sur une des images faites sur mesures pour gnlk. 

Les `LABEL` permettent d'ajouter des infos textuelles sur des images. Ici je m'en sert pour indiquer l'email du maintainer de l'image.

Les `ENV` sont des variables d'environnements qui persisteront sur tout le cycle de vie du conteneur depuis sa création dans le Dockerfile. C'est utile pour définir un `path` par exemple. Dans les images gnlk, on s'en sert pour indiquer le chemin de l'executable de Composer. 

Les `ARGS` sont des variables d'environnements qui ne persistent pas après la création de l'image. On s'en sert généralement pour gérer les versions d'outils pour faire de la génération flexible d'images. Par exemple, je m'en sert dans nos images gnlk pour - avec le même Dockerfile - générer toutes les images de PHP. 

## Les scripts (RUN, COPY, CMD)

Vous aurez très probablement des actions à effectuer dans vos images afin de faire marcher votre projet. Dans ce cas, vous utiliserez des `RUN`. Cela permet de lancer des lignes de commandes comme vous le feriez dans n'importe quelle console. Vous pouvez enchainer les commandes dans le même `RUN` en utilisant `&&`. Pour un meilleur formatage, on utilise généralement `\` pour faire un retour à la ligne. Le bon combo pour faire plusieurs commandes est donc de faire comme suit :  

```
RUN commande 1 && \
    commande 2
```

Vous aurez également besoin de copier des fichiers dans votre conteneur (différent des volumes de docker-compose). A titre de rappel, une image Docker doit être capable de tourner toute seule, sans docker-compose, ni aucun montage d'aucune sorte. Elle devra donc comporter vos fichiers ! Pour ce faire, vous pourrez utiliser `COPY`. 
Souvent, j'utilise l'instruction `COPY . .` qui veux dire "copie le dossier courant (.) dans le dossier courant sur le conteneur (.)". Ceci est possible car nous déclarons le `WORKDIR` (répertoire par défaut dans le conteneur) comme étant le dossier où seront nos fichiers. 
Il est également possible d'utiliser le paramètre `COPY --chown=X:X` qui permet d'assigner automatiquement l'utilisateur X et le groupe X au dossier concerné sur le conteneur. Si vous avez des notions de Linux vous comprendrez que c'est très utile vu que nous évitons dans la mesure du possible d'utiliser l'utilisateur `root`.

Pour terminer, il faudra définir le `CMD` de votre image. Concrètement, c'est ce qui va se passer lorsque vous allez lancer le conteneur. (Il est également possible d'utiliser un script et de l'indiquer via `ENTRYPOINT` mais c'est un sujet bien plus complexe). De base, nos images lancent `php-fpm` mais vous pouvez avoir besoin de faire plus de choses, comme un `composer install`, un `yarn install`, ou plein d'autres choses. 

# Bonnes pratiques et optimisations

Dans l'objectif d'avoir des images les plus légères, rapides et flexibles possible, merci de suivre ces quelques règles.

## Un RUN pour les gouverner tous

Il est important de savoir que chaque instruction ajouté dans un Dockerfile créer ce qu'on appel un `layer`. C'est un système de couche. Chaque `layer` ajoute du poids sur l'image ainsi que du temps de process. Pour réduire ce poids et pour accélerer le traitement des images, il est primordial de réduire au possible le nombre d'instructions que vous ajouter dans vos Dockerfile.

Évidemment, vous ne pourrez pas faire de miracle si vous devez lancer un `COPY`, un `RUN` et un `CMD`, mais vous pouvez cependant mettre toutes vos commandes linux dans le même `RUN` afin d'éviter d'en mettre plusieurs. C'est la principale voie d'optimisation. 

Pendant le dev, vous pouvez séparer vos `RUN`, car chaque modification de votre Dockerfile oblige Docker à recompiler l'instruction en question. Vous gagnerez donc du temps à ne pas tout recompiler. Mais au moment de passer en production, il est intéressant de tout regrouper.

## Une image autonome c'est du temps de gagné

Si vous observez les Dockerfile de nos projets, vous constaterez que nous effectuons un `composer install` dans un `RUN` mais **également** dans notre `CMD` de dev. Ce n'est pas un hasard !

Ce qu'il faut comprendre, c'est que tout ce qui se passe dans vos `RUN `sera *imprimé* dans la définition de votre image. Concrètement c'est ce qui permet de lancer votre image n'importe ou et n'importe comment et qu'elle fonctionne. Mais le fait de faire ça ne permet **que** d'avoir vos dépendances dans vos fichiers du dit conteneur. Comment doit-on faire pour le dev, où les fichiers doivent êtres présents en local pour qu'on puisse bosser avec ? C'est la que `CMD` entre en scène. 

Pour rappel, `CMD` est lancé au démarrage du conteneur et nous utilisons (en dev) un volume monté à l'emplacement du projet. Le fait de lancer `composer install` au démarrage du conteneur permettra donc de récupérer les dépendances dans votre dossier de dev vu qu'il est monté. De cette façon, vous travaillerez de manière transparente et le montage permettra de ne pas avoir à recompiler l'image à chaque modification.

Observez cependant que notre image de *dev* fait un `composer install --no-progress --no-suggest` qui installe toutes les dépendances, même celles de dev, alors que notre image de *prod* ne **fait pas** de `composer install`. Pourquoi ? Parce que dans un contexte de production (ou preproduction) nous n'avons pas besoin d'effectuer d'installation de dépendance à chaque démarrage de conteneur car :
- Cela voudrais dire que nous perdons une bonne minute à chaque démarrage d'image pour installer les dépendances
- Nous avons déjà les dépendances dans notre `RUN`. Inutile de le refaire.
- En prod, si un conteneur crash, il est automatiquement relancé. Pour des raisons de disponibilité, l'image doit être lancé le plus rapidement possible

## Ne jamais utiliser root

L'utilisateur `root` c'est pratique car on peux faire ce qu'on veux, mais c'est également un risque important pour la sécurité. Dans un conteneur c'est pareil. Même si l'on est "protégé" par l'isolation de notre conteneur, il existe des risques. Dans tout les cas, vous n'avez pas envie que par X ou Y moyen un utilisateur puisse récupérer l'accès à la console et foute le bordel dans votre application.

De plus, il existe des manipulations complexes qui permettent en utilisant root de remonter au delà du conteneur. 

Pour parer à ça, il suffit de créer un utilisateur dans notre Dockerfile puis de le définir comme utilisateur actif. Évidemment, on installe pas sudo sinon ça ne sert à rien !

Voici un exemple :  
```
RUN mkdir /home/appuser && \ → on créer le dossier perso de l'utilisateur
    chown 1000:1000 /home/appuser && \ → on défini l'utilisateur et le groupe du dossier
    groupadd -r -g 1000 appuser && \ → on créer le group en définissant l'ID à 1000
    useradd -r -u 1000 -b /home -g appuser appuser && \ → on créer l'utilisateur en définissant l'ID à 1000 et le group à appuser
    chown -R 1000:1000 /var/www/ → on définie les droits sur le dossier de notre appli
USER appuser → on définie l'utilisateur actif à partir de cette ligne
```

### Utilisation du group et user id 1000

Il faut savoir que sur WSl (et sur linux de manière globale) votre premier utilisateur courant aura pour ID 1000 et sera dans son propre groupe portant le même nom et avec le même ID.

Pour éviter d'avoir des problèmes de droits sur les fichiers lors de montages via docker-compose, on créer un utilisateur dans le Dockerfile qui possède le même ID que l'utilisateur spécifié.

C'est une bonne pratiques qui se démocratise et qui reste très pratique sans avoir aucun effet négatif alors ...

## Ne copier pas de fichiers inutiles

Pas besoin de rentrer beaucoup dans le détails ici, mais vous l'aurez compris, plus vous avez de fichiers dans votre conteneur, plus il sera lourd. Réduisez donc les fichiers copiés et ignorez ceux qui sont inutiles. Vous pouvez utiliser un fichier `.dockerignore` pour ce faire. (même fonctionnement que .gitignore)

# Différences entre Debian et Alpine pour vos scripts

## Gestion de l'utilisateur courant

Uniquement question de syntaxe ici.

Debian :  
```
RUN mkdir /home/appuser && \
    chown 1000:1000 /home/appuser && \
    groupadd -r -g 1000 appuser && \
    useradd -r -u 1000 -b /home -g appuser appuser && \
    chown -R 1000:1000 /var/www/
USER appuser
```

Alpine :   
```
RUN addgroup -S 1000 && adduser -S appuser -u 1000 -h /home/appuser -D -G 1000 && \
    chown -R 1000:1000 /var/www/
USER appuser
```

## Gestion des paquets

Il est question de syntaxe ici, mais il est également possible que le nom des paquets soit différent !

Debian :  
```
RUN apt-get update && \ → mettre à jour les repos
    apt-get install -qy pdftk → installer pdftk (-q = quiet -y = yes)
```

Alpine :  
```
RUN apk add --no-cache -u fcgi → installer fcgi (--no-cache = n'écris rien en cache (poids) -u = update (repos))
```

Pour Alpine, vous pourrez utiliser `apk update && apk search XXX` pour rechercher un paquet

## Scripts

Tout vos scripts devront être compatible avec `sh` (bash n'est pas installé et ne doit pas l'être). Si vous lancer une ligne de commande vous pourrez utiliser `sh -c "commande 1 && commande 2"`

