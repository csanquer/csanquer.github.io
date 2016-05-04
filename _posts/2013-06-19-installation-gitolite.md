---
layout: post
title:  "Installation Gitolite"
date:   2013-06-19 15:12:00 +0100
summary: Installation d'un serveur git avec gitolite
thumbnail:  heart
categories:
    - tutoriels
tags:
    - git
    - gitolite
---

Préparation
-----------
Au préalable, vous devez installer git et gitweb :

```sh
sudo aptitude install git gitweb
```


et créer un compte unix git qui va heberger gitolite et les dépots git :

```sh
sudo adduser --system --shell /bin/bash --gecos 'git version control' --group --disabled-password --home /home/git git
```

Ensuite vous devez copier votre clé publique ssh (pour administrer gitolite et utiliser git) sur le serveur dans `/home/git` et la renommer en `VotrePrenom.VotreNom.pub`

Installation de gitolite
------------------------

On supprime les eventuelles clés SSH déjà autorisées et on prepare le PATH de l'utilisateur git

```sh
sudo su git
rm $HOME/.ssh/authorized_keys
mkdir -p $HOME/bin
echo 'PATH=$HOME/bin:$PATH' >> $HOME/.bashrc
source .bashrc
```

puis on installe gitolite depuis [github](https://github.com/sitaramc/gitolite "github gitolite") :

```sh
git clone git://github.com/sitaramc/gitolite
cd gitolite
git co v3.5.3.1
cd ~
gitolite/install -ln $HOME/bin
gitolite setup -pk VotrePrenom.VotreNom.pub
```

Configuration de Gitweb pour gitolite
-------------------------------------

Gitweb ne fonctionne pas avec gitolite par defaut:
On va adapter sa configuration en remplaçant le contenu du fichier `/etc/gitweb.conf` par celui-ci :
