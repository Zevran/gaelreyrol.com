+++ 
date = 2017-05-08
title = "Gérer ses secrets avec Vault - 1ère partie"
slug = "gerer-ses-secrets-vault-part1" 
tags = ["vault", "secrets", "coffre-fort", "hashicorp", "startup", "entreprise"]
categories = ["sécurité"]
+++

## Introduction

La gestion des secrets, clés API et autres données sensibles est une question délicate et réccurente au sein d'une startup, d'une entreprise.

Je pense que beaucoup on eu à faire à ce cas de figure où certains secrets sont stockés dans un fichier au minimum chiffré (dans le cas contraire c'est grave), ce même fichier étant sur un stockage interne centralisé (voir externe) afin que tout le monde y ai accès.

Je pense que vous savez tout aussi bien que moi que cette pratique est à totalement éviter !

Mais comment est-il possible d'éviter cette pratique sans être obliger de déployer des moyens gigantesques ?

Comment permettre l'accès à des données sensibles de façon organisée et sécurisée à l'ensemble des développeurs et de façon programmatique ?

La réponse est [Vault](https://www.vaultproject.io/), créé par [HashiCorp](https://www.hashicorp.com) une société basée à San Francisco travaillant sur les problématiques liées au domaine [DevOps](https://fr.wikipedia.org/wiki/Devops). Ils ont d'ailleurs écrit un livre blanc sur le sujet, juste [ici](https://422bf3d160f51e6f9d81-50529851d44f252cc6434d6bbf378de4.ssl.cf2.rackcdn.com/DevOps-Defined.pdf).

Vault stocke et sécurise vos mots de passe, clés API, certificats et autres secrets par l'intermédiaire de jetons. Ces jetons permettent étroitement de contrôler, de révoquer et d'auditer l'accès aux données. Vault est aussi doté d'une API rendant la manipulation, le contrôle de ces données complètement programmatique donc facilement intégrable à n'importe quelle application ou processus automatisé.

Bon ça c'était juste l'introduction histoire de vous mettre dans le bain, maintenant on y met les mains pour que ce soit un petit peu plus concret.

## Pas à pas

### Installation

On commence par télécharger [Vault](https://www.vaultproject.io/downloads.html) en fonction de votre platforme, l'avantage c'est qu'il n'y a pas grand chose à faire puisque tout est embarqué dans un seul exécutable.

***Je travaille sous Mac Os donc pensez à adapter en fonction de votre OS :P***

```bash
mkdir learn-vault && cd learn-vault
wget https://releases.hashicorp.com/vault/0.7.0/vault_0.7.0_darwin_amd64.zip
unzip vault_0.7.0_darwin_amd64.zip
mv vault /usr/local/bin
```

Vous ne devriez obtenir qu'un seul fichier, le binaire vault.
Notez que ce binaire fait à la fois office de serveur et de client :)

### Lancement du serveur

La première chose à faire c'est de lancer le serveur vault en environnement de développement.
Attention lorsque vault est lancé en mode développement celui-ci stocke toutes les données dans la mémoire donc si vous l'arrêtez vous devrez tout recommencer !

```bash
$ vault server -dev
```

Vous devriez voir apparaître quelque chose comme ça :

```bash
==> WARNING: Dev mode is enabled!

In this mode, Vault is completely in-memory and unsealed.
Vault is configured to only have a single unseal key. The root
token has already been authenticated with the CLI, so you can
immediately begin using the Vault CLI.

The only step you need to take is to set the following
environment variables:

    export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are reproduced below in case you
want to seal/unseal the Vault or play with authentication.

Unseal Key: Cu4W9e1IcxEvEGoPMAaVgC84sFTS7f+PfXpyJyVQOUc=
Root Token: c67623a1-a907-a676-1cd0-3c34b9a8fe4c

==> Vault server started! Log data will stream in below:
```

Les informations importantes à retenir sont :

#### VAULT_ADDR
La variable d'environnement pour configurer le point d'accès API qui servira au client vault à échanger avec le serveur.
```bash
export VAULT_ADDR='http://127.0.0.1:8200'
```


#### Unseal Key
La clé de déverrouillage, une partie très importante du fonctionnement intrinsèque de vault permettant de chiffrer les données. En mode développement le vault est déjà déverrouillé.
```bash
Unseal Key: zYuoHnBukMnVxtpP0CSfSHw1sX2iUCUwlVPTuCAaBAU=
```

Pour résumer vault sait exactement comment accéder aux données mais ne sait abosulement pas comment les décrypter.

Donc au moment de son initialisation vault va créer une clé de chiffrement stockée avec les données (c'est elle qui va chiffrer vos données) mais elle-même chiffré avec une clé maitresse et il se trouve que cette clé n'est jamais stockée. C'est cette clé qui vous est restitué à l'initialisation de vault et qui vous permettra de verrouiller/déverrouiller vos données.

Si vous lancez vault en environnement de production cette clé pourra être fragmentée en plusieurs morceaux grâce au [partage de clé secrète de Shamir](https://fr.wikipedia.org/wiki/Partage_de_cl%C3%A9_secr%C3%A8te_de_Shamir) afin de pouvoir répartir ces morceaux entre différents développeurs par exemple. Ainsi si un poste est infecté, piraté où je ne sais quoi le vault n'est pas compromis car il faudra réunir ces différents morceaux pour déverrouiller le vault. L'avantage de ce système est que vous pouvez remplacer ces morceaux par des clés publiques [PGP](https://fr.wikipedia.org/wiki/Pretty_Good_Privacy) et ça c'est vraiment top !


#### Root Token
Le jeton 'root' vous servira à configurer le vault et manipuler vos données, les jetons 'root' permettent tout, vraiment tout. En mode développement vous êtes déjà authentifié en tant que root.
```bash
Root Token: 1cce6c60-8878-cafe-fbe6-8d4a275f318c
```

Dans ce pas à pas nous allons l'utiliser pour passer en revue les principales fonctionnalités de vault mais il n'est abosulement pas recommandé de l'utiliser en production. Je vous expliquerai dans un autre article comment il est possible de s'en débarasser.

### Écrire votre premier secret
Cela se fait très simplement en utilisant la commande ```write``` comme ci-dessous :
```bash
$ vault write secret/johnsnow isdead=askmelisandre
Success! Data written to: secret/johnsnow
```
Cette commande se décompose en deux parties, ```secret/johnsnow``` le chemin d'accès aux données et ```isdead=askmelisandre``` la combinaison clé valeur correspondant aux données à stocker. Nous verrons un peu plus tard pourquoi le chemin est préfixé par ```secret```.

Vous pouvez accumuler autant de combinaisons clé valeur que vous souhaitez. Cependant faites attention l'éxécution de cette commande sur un même chemin écrase les données précédentes.

La commande vous permez aussi d'écrire des données venant de l'entrée standard [STDIN](https://fr.wikipedia.org/wiki/Flux_standard) ou encore venant d'un fichier.

### Lire votre secret
Et bien c'est aussi simple que l'écriture :
```bash
$ vault read secret/johnsnow
Key             	Value
---             	-----
refresh_interval	768h0m0s
isdead          	askmelisandre
```

Vous pouvez aussi demander une réponse en json avec l'option ```-format=json``` et la lire avec [jq](https://stedolan.github.io/jq/) par exemple
```bash
$ vault read -format=json secret/johnsnow
{
	"request_id": "ad311978-48f7-0fae-2223-6aaf19b8d1fc",
	"lease_id": "",
	"lease_duration": 2764800,
	"renewable": false,
	"data": {
		"isdead": "askmelisandre"
	},
	"warnings": null
}
$ vault read -format=json secret/johnsnow | jq -r .data
{
  "isdead": "askmelisandre"
}
```

### Supprimer votre secret
Même logique ici, cependant faites très attention car la suppréssion est définitive.
```bash
vault delete secret/johnsnow
Success! Deleted 'secret/johnsnow' if it existed.
```

---

Je vous prie de m'excuser pour cet arrêt en cours de route mais le sujet étant
particulièrement velu je me suis aperçu qu'il serait plus facile
à comprendre pour vous et à faire pour moi en plusieurs parties :D.

La suite arrivera je pense d'ici une semaine ou deux !

N'hésitez pas à réagir à cet article dans les commentaires ou par mail :)
