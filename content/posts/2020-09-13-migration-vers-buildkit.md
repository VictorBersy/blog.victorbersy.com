---
title: "Réduire le temps de build Docker grâce à BuildKit"
date: 2020-09-13T13:00:09+02:00
draft: true
featured_image: '5279ab2f63c6dc4856cb699e50080011.png'
---

## Contexte

Chez Selectra, nous utilisons Docker sur tous nos environnements: du local à la production en passant par les pipelines de CI.

Nous avons de nombreux projets web actif, et comme la plupart de ces projets, ils nécessitent l'installation de dépendances:
- Paquets PHP via composer
- Paquets JS via NPM
- Extensions PHP

Ces étapes prennent pas mal de temps, donc utiliser du cache devient rapidement une nécessité pour garder des temps de CI raisonnables.

Pour obtenir des images plus légères, j'utilise les build multi-stage, dont voici la structure:

```dockerfile
FROM composer:2 as composer-build
...

FROM node:10-alpine as npm-build
...

FROM php:7.3-apache as php-extensions-build
...

# Final build, "app", our base for others
FROM php:7.3-apache as app
...

# Various stages for release/dev/etc...
# Release stage, what will be deployed on our servers
FROM app as release
...

# Back dev stage, install backend dev tools here
FROM app as dev-back
...

# Front dev stage, install frontend dev tools here
FROM npm-build as dev-front
...

# Test stage, used during CI/CD and local testing
FROM app as test
...

# Default docker build target will be 'app'
FROM app
```

Sans Docker, il suffirait de cacher ces dossiers:
- `/vendor` pour les paquets PHP
- `/node_modules` pour les paquets JS
- Le dossier où les extensions PHP s'installent

Seulement voilà, avec Docker, c'est un peu plus compliqué. Ces dossiers étant sur le container, il est impossible de les faire remonter au host pour les cacher.

## Problème : difficile de cacher tous les stages

La solution qui semble la plus simple à première vue, est de build et tagguer chaque stage indépendamment:
```bash
docker build -t mon-projet:composer-build --target=composer-build .
docker build -t mon-projet:npm-build --target=npm-build .
docker build -t mon-projet:php-extensions-build --target=php-extensions-build .
# etc...
```

Et de tout envoyer sur un registre de container tel que AWS ECR:
```bash
docker push aws_account_id.dkr.ecr.region.amazonaws.com/mon-projet:composer-build
docker push aws_account_id.dkr.ecr.region.amazonaws.com/mon-projet:npm-build
docker push aws_account_id.dkr.ecr.region.amazonaws.com/mon-projet:php-extensions-build
# etc...
```

Il suffit ensuite de pull chaque image avant de commencer les builds pour que le cache soit actif.

Seulement voilà, on se retrouve à maintenir une liste explicite des stages qu'on veut build et cacher et on doit uploader chaque image sur un registre manuellement.

Un processus long et fastidieux dont on se passerait bien.

## Problème : AWS ECR est trop lent pour cacher nos stages

Un autre problème se pose si nous optons pour cette solution: AWS ECR n'est pas des plus rapides quand il s'agit d'uploader des images dessus. Envoyer de nombreuses images Docker dessus va prendre un temps non négligeable.

## Problème : Difficile de build en parallèle ce qui peut l'être

Par exemple, nous pourrions build `composer-build` et `npm-build` en parallèle comme ces deux stages sont totalement indépendants.

Malheureusement, ça impliquerait de faire ça nous même avec un peu de bash, et de maintenir également ces dépendances manuellement, alors qu'elles sont déjà décrites dans le Dockerfile.

## Solution : BuildKit

Grâce au projet BuildKit, nous pouvons facilement exporter cache et image en une seule commande:

En précisant la cible (`test`), on garantit également de build uniquement ce dont on a besoin pour nos tests:
```bash
buildctl build ... \
  --output type=image,name=localhost:5000/myrepo:image,push=true \
  --export-cache type=registry,ref=localhost:5000/myrepo:buildcache,push=true \
  --import-cache type=registry,ref=localhost:5000/myrepo:buildcache \
  --opt target=test
```

## Solution : Registre de container privé par SemaphoreCI

Je ne sais pas si d'autres services de CI fournissent également ce service, mais SemaphoreCI a eu la bonne idée d'inclure un registre privé d'image Docker.

Les temps d'upload sont excellents, bien meilleur que sur AWS ECR.

## Avant / Après + screenshots

## Conclusion
