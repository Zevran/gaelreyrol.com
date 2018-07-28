+++ 
date = 2017-09-17
title = "Démon OpenVPN sur Ubuntu"
slug = "daemon-openvpn-ubuntu" 
tags = ["openvpn", "daemon", "ubuntu", "systemctl"]
categories = ["openvpn"]
+++

## Comment lancer un démon client OpenVPN sur Ubuntu

Oui je sais c'est mon premier tutoriel de la rentrée et celui-ci n'a plus ou moins rien avoir avec le DevOps.
Je me suis décidé à faire ce tutoriel parce que j'ai remarqué qu'il y avait beaucoup de tutoriels sur comment
créer un serveur [VPN](https://en.wikipedia.org/wiki/Virtual_private_network) mais quasiment aucun sur comment se connecter à ceux-ci et encore moins depuis un serveur dédié ou sans GUI.

De plus j'ai quelques collègues qui travaillent sous Ubuntu et qui auraient bien eu besoin d'un petit tuto comme celui-ci.

Ce tutoriel est à destination de la distribution [Ubuntu](https://www.ubuntu.com/) avec comme serveur VPN la solution [OpenVPN](https://openvpn.net/).

Si vous ne disposez pas d'un serveur OpenVPN auquel vous connecter mais que vous disposez d'une petite [Raspberry Pi](https://www.raspberrypi.org/) je vous recommande le site [PiVPN](http://www.pivpn.io/) pour créer votre propre serveur.

Travaillant sur Mac OS, j'utilise l'outil génial qu'est [Vagrant](https://www.vagrantup.com/) pour disposer d'une machine virtuelle sous Ubuntu, si vous êtes sous Ubuntu vous pouvez bien évidemment passer les quelques étapes qui vont suivre.

#### Installer Vagrant

Pour cela rien de plus simple que se rendre sur la page des [téléchargements](https://www.vagrantup.com/downloads.html) et de récupérer l'installateur correspondant au système d'exploitation de votre machine.

#### Installer VirtualBox

Vagrant c'est super mais tout seul il ne fera rien, pour pouvoir l'utiliser correctement il nous faut un logiciel de virtualisation, perso je prends [VirtualBox](https://www.virtualbox.org/) parce que c'est gratuit et que ça fait le taff mais vous pouvez tout aussi bien choisir d'autres [fournisseurs](https://www.vagrantup.com/intro/getting-started/providers.html) comme VMware ou encore AWS.

Donc même chose ici, il suffit de se rendre sur la page des [téléchargements](https://www.virtualbox.org/wiki/Downloads) et de récupérer l'installateur qui convient.

#### Créer un machine virtuelle Ubuntu

On ouvre une fenêtre de terminal, on se place dans le répertoire de notre choix et on lance la commande suivante.
```bash
vagrant init ubuntu/xenial64
```

Cette commande configure une machine virtuelle sous Ubuntu Server 16.04 LTS.
Je vous passe les explications sur Vagrant et ses fonctionnalités parce que ça fera l'objet d'un article.

#### Lancement de la machine virtuelle

Up, c'est comme le Port Salut c'est écrit ....
```bash
vagrant up
```

Vagrant va donc télécharger l'image correspondante et lancer la machine virtuelle.


#### Connexion à la machine virtuelle

C'est comme le Port Salut ... ok je sors -> [].
```bash
vagrant ssh
```

Nous sommes donc maintenant connecté à une machine virtuelle Ubuntu prête à servir !

#### Installation d'OpenVPN

Si nous voulons pouvoir nous connecter à un serveur OpenVPN, il faut installer le paquet correspondant.

```bash
sudo apt-get update
sudo apt-get install openvpn
```

L'installation nous permet de disposer de l'exécutable `openvpn` et du répertoire `/etc/openvpn`. L'exécutable sert aussi bien de serveur que de client.

C'est dans le répertoire `/etc/openvpn` que nous allons configurer notre client OpenVPN.

#### Configuration du client

Vous devez normalement avoir à disposition un fichier de configuration OpenVPN qui vous permettra de vous connectez à votre serveur en fonction de ses configurations.

Le mien ressemble à ça :
```
client
dev tun
proto udp
remote 192.168.0.10 1194
dhcp-option DNS 192.168.0.1
resolv-retry infinite
nobind
persist-key
persist-tun
key-direction 1
remote-cert-tls server
tls-version-min 1.2
verify-x509-name server name
cipher AES-256-CBC
auth SHA256
comp-lzo
verb 1
<ca>
Le contenu d'un certificat au format PEM
</ca>
<cert>
Le contenu d'un certificat au format PEM
</cert>
<key>
Le contenu d'une clé privée au format PEM
</key>
```

Copiez le contenu de votre fichier de configuration dans un fichier sur votre poste, serveur ou machine virtuelle dans le répertoire `/etc/openpvn` puis testez le. Il faut absolument que l'extension de votre fichier soit en `.conf` :)

```bash
sudo openvpn --config /etc/openvpn/home.conf
```

Vous devriez pouvoir vous connecter sans soucis, si jamais c'est pas le cas. Jetez un oeil aux logs, l'erreur est généralement explicite.

Pour ma part c'est une authentification par clé privée, je n'ai qu'à saisir le mot de passe de ma clé pour valider la connexion.

Je vérifie que mon interface `tun` est bien montée.

```bash
sudo ifconfig tun0
```

Paf ça passe bien, youhou !

Sauf que voilà, disons que j'ai un serveur DNS dans le même réseau local que mon serveur VPN et que je souhaitrai pouvoir accéder à des domaines locaux du type `git.local` ou `cozy.local`.

Et bien je ne peux pas parce que les serveurs de noms n'ont pas été mis à jour au montage de l'interface `tun0`. Je pourrais éditer le fichier `/etc/resolv.conf` mais c'est mal m'voyez ? Et puis c'est écrit dessus comme le Port Salut ...

```
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
```

Heureusement pour nous, Ubuntu a pensé à joindre un ficher à l'installation d'OpenVPN, c'est le fichier `update-resolv-conf` qui se trouve dans le répertoire `/etc/openvpn`.

Il va nous permettre de mettre à jour les serveurs de noms au montage et au démontage de l'interface `tun0`. Pour cela il nous suffit de rajouter dans notre fichier de configuration les lignes suivantes.

```
up /etc/openvpn/update-resolv-conf
down /etc/openvpn/update-resolv-conf
```

Et la ligne suivante si elle n'est pas déjà présente pour permettre l'exécution du fichier `update-resolv-conf`.

```
security-script 2
```

Maintenant que notre fichier configure correctement les serveurs de noms, il va nous falloir fournir les identifiants de connexion au serveur VPN.

*WOW WAIT* - Je sais ça peut paraître bizarre et à première vue assez risqué d'écrire en dur ses identifiants mais c'est le seul moyen pour permettre au démon de se connecter au serveur. Il ne faut juste pas oublier certaines précautions d'usage.

Mon serveur VPN est configuré pour s'authentifier par clé donc je vais devoir rajouter la ligne suivante dans mon fichier.

```
askpass /etc/openvpn/home-creds.txt
```

Je passe en paramètre à l'option `askpass` le chemin vers le fichier contenant le mot de passe de ma clé. N'oubliez pas de créer ce fichier et d'y inscrire votre mot de passe.

Si vous devez vous connecter avec un nom d'utilisateur et un mot de passe, il faudra replacer l'option `askpass` par `auth-user-pass` et remplacer le contenu du fichier correspondant par votre nom d'utilisateur sur la première ligne et votre mot de passe sur la deuxième.

N'oublier pas de changer les droits d'accès au fichier avec la commande `chmod`.

```bash
sudo chmod 400 /etc/openvpn/home-creds.txt
```

Si on essaye de se reconnecter avec la même commande.

```bash
sudo openvpn --config /etc/openvpn/home.conf
```

BOUM, plus besoin de saisir un mot de passe ou des identifiants.

#### Lancement du démon

Maintenant que notre configuration est fin prête nous pouvons lancer le client OpenVPN en démon via un service. Eh oui encore une fois Ubuntu a eu la bonne idée de fournir un script de type [upstart](https://doc.ubuntu-fr.org/upstart).

Pour lancer notre démon il suffit juste de démarrer le service OpenVPN accompagné du nom de notre fichier de configuration.

```bash
sudo service openvpn@home start
```

Si jamais vous n'arrivez pas à vous connecter, vous pouvez accéder aux logs du démon avec `journalctl` pour essayer de trouver la source de l'erreur.

```bash
sudo journalctl -u openvpn@home.service
```

*BONUS*

Puisque c'est une service, on peut très facilement l'activer pour qu'il se lance au démarrage de la machine avec la commande `systemctl`.

```bash
sudo systemctl enable openvpn@home.service
```

Et voilà, j'espère que ça vous a plus et que ça vous sera utile surtout !

Encore une fois n'hésitez pas à me faire part de vos avis ou problèmes dans les commentaires je serais très heureux de pouvoir y répondre !