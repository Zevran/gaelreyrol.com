+++ 
date = 2017-06-07
title = "Authentification & Polices avec Vault - 2ème partie"
slug = "authentification-polices-vault-part2" 
tags = ["vault", "secrets", "coffre-fort", "hashicorp", "startup", "entreprise"]
categories = ["sécurité"]
+++


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
d'authentification afin de contrôler l'accès aux secrets via des polices.

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

Pour ajouter un utilisateur il faudra créer celui-ci de la même manière qu'un secret sauf qu'au lieu d'écrire dans un chemin commençant par ```secret/``` il faudra écrire dans ```auth/userpass/users```. Vous pouvez obtenir des informations supplémentaires sur ce chemin en utilisant la commande ```vault path-help```.

Vous allez donc créer votre utilisateur en renseignant son nom d'utilisateur à la fin du chemin, son mot de passe et les polices en paramètres.
```bash
$ vault write auth/userpass/users/gaelreyrol password=lovedevops policies=devops
```
```bash
Success! Data written to: auth/userpass/users/gaelreyrol
```
Hop, notre utilisateur est fin prêt ! En revanche il nous manque la police ***devops*** qui va nous servir à définir des règles d'accès aux secrets.

#### Création d'une police

Les polices dans Vault permettent de définir des droits très précis sur les secrets. Les secrets sont construits sous forme d'arbre où chacun des noeuds est délémité par un ***/***, nous allons donc pouvoir appliquer des restrictions/permissions à ceux-ci en fonction d'une police.

Nous avons à disposition 7 capacités décrivant toutes les actions possibles :
- create: Création d'un secret.
- read: Lecture d'un secret.
- update: Modification d'un secret.
- delete: Suppression d'un secret.
- list: Énumération des secrets pour un chemin précis.
- sudo: Donne accès à des chemins protégés par des droits root.
- deny: Ne permet aucune action.

Nous pouvons aussi définir des paramètres permis et non permis :
- allowed_parameters
- denied_parameters

Ils permettent tout deux via un tableau d'élément de définir les clés et valeurs pouvant être modifiées, créées, supprimées ou non. Le petit bonus étant qu'ils supportent l'englobement en suffixe et préfixe.

Enfin la dernière fonctionnalité liée aux polices est le TTL maximum et minimum via les attributs :
- min_wrapping_ttl
- max_wrapping_ttl

Ils permettent de définir une durée vie minimum et maximum englobate des clés comprises par la police afin de forcer un élément à disparaitre ou d'être sur que d'autres éléments ne vont pas disparaître prématurément.

Nous allons maintenant pouvoir créer notre première police !
Pour cela nous allons créer un fichier ***devops.hcl***.

Petite précision, un fichier [HCL](https://github.com/hashicorp/hcl) est un fichier de configuration HashiCorp dérivé du JSON.

```
path "secret/apps/*" {
  capabilities = ["read", "create", "update", "list"]
}

path "secret/devops" {
  capabilities = ["create"]
  allowed_parameters = {
    "*" = []
  }
  denied_parameters = {
    "root" = []
  }
}
```

Nous voici donc avec une police qui permet la lecture, la création, la mise à jour et l'énumération sur tout ce qui sera compris dans le chemin ***secret/apps/***. Ainsi que la capacité à créer des secrets uniquement au niveau du chemin ***secret/devops*** avec n'importe quel nom de clé possible sauf ***root***.

Maintenant il ne nous reste plus qu'à intégrer cette police dans vault. Pour cela nous allons utiliser la commande ***vault policy-write*** qui prend en premier argument le nom de la police et en deuxième le chemin vers le fichier.

```bash
$ vault policy-write devops ./devops.hcl
```
```
Policy 'devops' written.
```

Vous pouvez à tout moment lister les polices dans vault via ```vault policies``` et afficher un police en particulier en ajouter son nom en argument.

#### Mise à l'épreuve

Maintenant que nous avons créer notre utilisateur et notre police, nous pouvons enfin tester si notre police fonctionne correctement.

Tout d'abord nous devons nous authentifier avec l'utilisateur créé plus haut, pour cela il nous faut utiliser la commande ```vault auth``` suivit d'un argument ```-method``` permettant de préciser notre méthode d'authentification ainsi que d'un argument ```-username```pour préciser l'utilisateur en question. Il vous faudra renseigner le mot de passe inscrit plus haut.

```bash
$ vault auth -method=userpass username=gaelreyrol
Password (will be hidden):
```
```bash
Successfully authenticated! You are now logged in.
The token below is already saved in the session. You do not
need to "vault auth" again with the token.
token: a8fdca6b-08a6-225d-95ac-2a0dbb405643
token_duration: 2764799
token_policies: [default devops]
```

À l'issue de cette authentification nous voyons bien que Vault nous a attaché à la police devops au niveau du champ ***token_policies***.

Pour tester notre police il nous suffit juste d'effectuer des actions au niveau des chemins explicités dans notre fichier car Vault applique la règle de liste noire. Tout ce qui n'est pas clairement défini est refusé par défaut.

Quelques exemples:
```bash
$ vault write secret/apps/postgres user=monapp password=$(openssl rand -base64 12)
$ vault read secret/apps/postgres
$ vault list secret/apps
$ vault delete secret/apps/postgres
$ vault list secret/devops
$ vault write secret/devops apps=[postgres]
$ vault read secret/devops
$ vault write secret/devops root=test
```

Je vous laisse le soin de mettre à l'épreuve Vault et de jouer avec les polices, capacités et paramètres.

---

Ce deuxième tutoriel sur Vault est terminé, j'espère qu'il vous sera utile et que vous saurez vous projeter dans une utilisation professionnelle de cet outil incroyablement puissant.

Le prochain tutoriel Vault portera sur son déploiement en production mais ça ne sera probablement pas le prochain article.

N’hésitez pas à partager et réagir à cet article dans les commentaires ou par mail :)