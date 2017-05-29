---
layout: post
title: "Authentification & ACLs avec Vault - 2ème partie"
comments: true
description: "Comment gérer l'accès aux secrets via des polices avec Vault"
keywords: "vault, secrets, coffre-fort, startup, entreprise, polices, acls, part 2"
cover: /assets/images/2017-05-08/vault.png
---

## Introduction

C'est parti pour un second tutoriel !

Dans le guide précédent j'ai
passé en revue les grands principes de Vault qui font de lui un outil sûr et adéquat
pour gérer ses secrets en entreprise ainsi que les étapes de base nécessaire à son
utilisation en terminant par la création, la lecture et la suppression d'un secret.
Si vous ne connaissez pas Vault et que vous n'avez pas lu la partie précédente, je vous
conseille d'aller y faire un tour pour démarrer sur de bonnes bases dans ce guide.

[Gérer ses secrets avec Vault - 1ère partie](https://gaelreyrol.com/2017/05/08/gerer_ses_secrets_vault)

Dans ce tutoriel nous allons voir comment il est possible de lier Vault à un système
d'authentification afin de contrôler l'accès aux secrets via des ACLs.

## Pas à Pas

### Les "Backends d'authentification"

Les ***backends d'authentification*** sont les modules permettant au client Vault de s'authentifier
et de s'assigner une identité afin d'accèder aux secrets selon des polices bien précises.

Il existe différents backends répondant à des besoins différents. Par exemple on utilisera plus facilement le backend [LDAP](https://www.vaultproject.io/docs/auth/ldap.html) pour gérer l'accès utilisateur si votre entreprise dispose d'un annuaire LDAP. En revanche on privilégira le backend
[AppRole](https://www.vaultproject.io/docs/auth/approle.html) pour un accès programmatique ou machine.

Voici la liste des différents [backends d'authentification](https://www.vaultproject.io/docs/auth/index.html).

Pour les besoins du tutoriel nous utiliserons le backend [Username & Password](https://www.vaultproject.io/docs/auth/userpass.html), il nous permettera de créer facilement des utilisateurs enregistrés sur le serveur Vault.

### Configuration du backend "Username & Password"

N'oubliez pas de démarrer un serveur local Vault en mode développement, les étapes qui vont suivre ne doivent surtout pas être exécutées sur un serveur Vault en production. Si vous ne savez pas comment faire, référez-vous au [premier tutoriel](https://gaelreyrol.com/2017/05/08/gerer_ses_secrets_vault) de cette série.

#### Activation du backend

Pour activer le backend ***Username & Password***, vous devez d'abord être authentifié en tant que ***root***. Vérifiez que vous êtes bien authentifié en tant que tel avec la commande :

```bash
$ vault token-lookup
```

Vous devriez voir apparaître au niveau de la clé ***policies*** la valeur ***root***.

```bash
Key             	Value
---             	-----
accessor        	15a5099a-19e2-cfd0-86b5-5ec51ddbbf51
creation_time   	1496092095
creation_ttl    	0
display_name    	root
expire_time     	<nil>
explicit_max_ttl	0
id              	41e9f79a-78b6-4889-0886-1683e983bf3c
meta            	<nil>
num_uses        	0
orphan          	true
path            	auth/token/root
policies        	[root]
ttl             	0
```

Si ce n'est pas le cas, arrêtez le serveur Vault, supprimez le fichier ```.vault-token``` à la racine de votre ```$HOME```, relancez votre terminal et redémarrez le serveur.

Vous pouvez maintenant activer le backend en utilisant la commande ```vault auth-enable```.
```bash
$ vault auth-enable userpass
```

Vault vous informera que vous le backend a été activé avec succès.
```bash
Successfully enabled 'userpass' at 'userpass'!
```

À noter que Vault a activé ***userpass*** sur le point de montage ***userpass***, cela veut dire que vous pouvez monter plusieurs même backends d'authentification mais avec un point de montage différent en utilisant l'option ```-path=<path>```.

Pour lister les méthodes d'authentification vous pouvez pouvez utiliser la commande ```vault auth -methods```. Vous devriez voir apparaître la méthode ***userpass*** :

```bash
Path       Type      Default TTL  Max TTL  Replication Behavior  Description
token/     token     system       system   replicated            token based credentials
userpass/  userpass  system       system   replicated
```

#### Création d'un utilisateur

Pour ajouter un utilisateur il vous faudra créer celui-ci de la même manière qu'un secret sauf qu'au lieu d'écrire dans un chemin commençant par ```secret/``` il vous faudra écrire dans ```auth/userpass/users```. Vous pouvez obtenir des informations supplémentaires sur ce chemin en utilisant la commande ```vault path-help```.


