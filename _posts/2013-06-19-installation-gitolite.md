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

Configuration du serveur Web (apache ou nginx)
----------------------------------------------

D'abord, ajoutez l'utilisateur apache (ou nginx) `www-data` au groupe `git`:

```sh
sudo adduser www-data git
```

puis créez un fichier vhost pour gitweb ( ex git.mydomain.example)

### Configuration Apache 2

créez le fichier `/etc/apache2/sites-available/gitweb-ssl` :

```
<IfModule mod_ssl.c>
<VirtualHost *:443>
  ServerName git.mydomain.example
  DocumentRoot "/usr/share/gitweb"
  DirectoryIndex index.cgi

  <Location />
      # try anonymous access first, resort to real
      #Satisfy Any
      # authentication if necessary.
      Require valid-user

      SSLRequireSSL

      # how to authenticate a user
      AuthType Basic
      AuthName "Gitweb : Depots git"
      AuthUserFile /home/git/git_users.passwd
   </Location>

   <Directory /usr/share/gitweb>
      Options FollowSymLinks +ExecCGI
      AddHandler cgi-script .cgi
   </Directory>

   CustomLog /var/log/apache2/gitweb.access.log combined
   ErrorLog /var/log/apache2/gitweb.error.log

   SSLEngine on
   SSLCertificateFile    /etc/ssl/certs/<my-ssl-certificate-pem>.pem
   # Add this once there is a real (non self-signed) certificate.
   SSLCertificateKeyFile /etc/ssl/private/<my-ssl-certificate-key>.key
</VirtualHost>

<VirtualHost *:80>
  ServerName git.mydomain.example

  Redirect / https://git.mydomain.example/
</VirtualHost>
</IfModule>
```

Enfin activez le vhost et redémarrez apache :

```sh
a2enmod rewrite actions headers ssl vhost_alias
a2ensite gitweb-ssl
service apache2 restart
```

### Configuration Nginx

créez le fichier `/etc/nginx/sites-available/gitweb-ssl` :

```nginx
server {
    listen 80 ;

    server_name git.mydomain.example;
    rewrite ^ https://git.mydomain.example$request_uri permanent;
}

# HTTPS server
#
server {
    listen 443;
    server_name git.mydomain.example;

    root /usr/share/gitweb;

    ssl on;
    ssl_certificate /etc/ssl/localcerts/<my-ssl-certificate-pem>.pem;
    ssl_certificate_key /etc/ssl/localcerts/<my-ssl-certificate-key>.key;

    ssl_session_timeout 5m;

    ssl_protocols SSLv3 TLSv1;
    ssl_ciphers ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv3:+EXP;
    ssl_prefer_server_ciphers on;

    access_log /var/log/nginx/gitweb.access.log;
    error_log /var/log/nginx/gitweb.error.log;
    charset utf-8;

    auth_basic           "RESTRICTED ACCESS";
    auth_basic_user_file /home/git/git_users.passwd;

    try_files $uri @gitweb;
    location @gitweb {
        fastcgi_pass unix:/var/run/fcgiwrap.socket;
        fastcgi_param SCRIPT_FILENAME   /usr/share/gitweb/gitweb.cgi;
        fastcgi_param PATH_INFO         $uri;
        fastcgi_param GITWEB_CONFIG     /etc/gitweb.conf;
        fastcgi_param REMOTE_USER     $remote_user;
        include fastcgi_params;
   }
}
```

Enfin activez le vhost et redémarrez nginx :

```sh
ngxensite gitweb-ssl
# ou ln -s /etc/nginx/sites-available/gitweb-ssl /etc/nginx/sites-enabled/gitweb-ssl
service nginx restart
```
