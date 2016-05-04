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

```perl
# debian gitweb conf with manual gitolite install
# handle gitolite 3 acl in gitweb

# path to git projects (<project>.git)
$projectroot = "/home/git/repositories";

# directory to use for temp files
$git_temp = "/tmp";

# target of the home link on top of all pages
#$home_link = $my_uri || "/";

# html text to include at home page
#$home_text = "indextext.html";

# file with project list; by default, simply scan the projectroot dir.
#$projects_list = $projectroot;

# stylesheet to use
#@stylesheets = ("static/gitweb.css");

# javascript code for gitweb
#$javascript = "static/gitweb.js";

# logo to use
#$logo = "static/git-logo.png";

# the 'favicon'
$favicon = "static/git-favicon.png";

# ----------------------------------------------------------------------

# Per-repo authorization for gitweb using gitolite v3 access rules
# Read comments, modify code as needed, and include in gitweb.conf

# Please note that the author does not have personal experience with gitweb
# and does not use it.  Some testing may be required.  Patches welcome but
# please make sure they are tested against a "github" version of gitolite and
# not an RPM or a DEB, for obvious reasons.

# ----------------------------------------------------------------------

# First, run 'gitolite query-rc -a' (as the gitolite hosting user) to find the
# values for GL_BINDIR and GL_LIBDIR in your installation.  Then use those
# values in the code below:

BEGIN {
    $ENV{HOME} = "/home/git";   # or whatever is the hosting user's $HOME
    $ENV{GL_BINDIR} = "/home/git/bin";
    $ENV{GL_LIBDIR} = "/home/git/bin/lib";
}

# Pull in gitolite's perl API module.  Among other things, this also sets the
# GL_REPO_BASE environment variable.
use lib $ENV{GL_LIBDIR};
use Gitolite::Easy;

# Set projectroot for gitweb.  If you already set it earlier in gitweb.conf
# you don't need this but please make sure the path you used is the same as
# the value of GL_REPO_BASE in the 'gitolite query-rc -a' output above.
#$projectroot = $ENV{GL_REPO_BASE};

# Now get the user name.  Unauthenticated clients will be deemed to be the
# 'gitweb' user so make sure gitolite's conf file does not allow that user to
# see anything sensitive.
#
# for nginx add this to nginx config
# fastcgi_param   REMOTE_USER     $remote_user;
#
$ENV{GL_USER} = $cgi->remote_user || "gitweb";

$export_auth_hook = sub {
    my $repo = shift;
    # gitweb passes us the full repo path; we need to strip the beginning and
    # the end, to get the repo name as it is specified in gitolite conf
    return unless $repo =~ s/^\\Q$projectroot\\E\\/?(.+)\\.git$/$1/;

    # call Easy.pm's 'can_read' function
    return can_read($repo);
};

$feature{'highlight'}{'default'} = [1];
$feature{'highlight'}{'override'} = 1;

# git-diff-tree(1) options to use for generated patches
#@diff_opts = ("-M");
@diff_opts = ();
```

Puis il faut configurer gitolite pour gerer les mots de passe pour gitweb et les droits UNIX sur les dépots.
Il suffit de modifier le fichier `/home/git/.gitolite.rc` en ajoutant ou modifiant les variables de config suivante:

```perl
%RC = (
    # ....
    UMASK                       =>  0027, #les depots doivent pouvoir être lu par le group git
    GIT_CONFIG_KEYS             =>  'gitweb\\..*', #
    HTPASSWD_FILE               =>  '/home/git/git_users.passwd', # pour pouvoir editer son mot de passe gitweb
    # ....
    COMMANDS                    =>
        {
            # ....
            'htpasswd'          =>  1, # pour pouvoir editer son mot de passe gitweb
            # ....
        },
    # ....
);
```

On change les droits des dépots (`drwxr-x---` pour les dossiers et `-rw-r-----` pour les fichiers ) afin qu'ils soient lisibles et modifiable par l'utilisateur git et lisibles uniquement par le groupe git.

```sh
chmod -R u=rwX,g=rX,o= /home/git/repositories/*
```

on crée un nouveau fichier de mot de passe pour gitweb (via la commande apache `htpasswd`)

```sh
htpasswd -c /home/git/git_users.passwd VotrePrenom.VotreNom
```
