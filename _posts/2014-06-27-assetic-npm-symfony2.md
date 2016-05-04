---
layout: post
title:  "Symfony2 avec npm, bower et assetic"
date:   2014-06-27 13:34:00 +0100
summary: Gestion des dépendances CSS, Javascript d'un projet Symfony2 avec npm, bower et assetic
thumbnail: cogs
categories:
    - tutoriels
tags:
    - symfony
    - assets
    - css
    - javascript
    - bower
    - assetic
---



Petit tutoriel pour combiner [composer](https://getcomposer.org), [npm](https://www.npmjs.org/) ([nodejs](http://nodejs.org/)), [bower](http://bower.io/) et [assetic](http://symfony.com/fr/doc/current/cookbook/assetic/index.html) afin de gérer toutes vos dépendances CSS, JS, PHP sur un projet symfony2.

### Prérequis

Vous avez besoin de :
* PHP >= 5.3 (évidement)
* nodejs >= 0.10
* Git pour le téléchargement des dépendances
* [composer](https://getcomposer.org)

### Configurer Composer

Il faut ajouter la commande `npm install` au niveau des hooks post-install et post-update de la section scripts du fichier `composer.json`

```json
{
    "scripts": {
        "post-install-cmd": [
            "npm install"
        ],
        "post-update-cmd": [
            "npm install"
        ]
    },
}
```

### Configurer NPM

Créez un fichier `packages.json` (vous pouvez utiliser notamment la commande `npm init`).
Le point important est de renseigner la section scripts - postinstall avec la commande bower.

```json
{
    "name": "my-sf2-project",
    "author": "Martin Durand martin.durand@example.com",
    "private": true,
    "dependencies": {
        "less": "~1.7",
        "uglify-js": "~2.4",
        "uglifycss": "~0.0",
        "bower": "~1.3"
    },
    "scripts": {
        "postinstall" : "node_modules/.bin/bower install"
    }
}
```

Ici, NPM est configuré de la façon suivante :

* l'application symfony2 est **privée** (elle ne doit pas être publié sur www.npmjs.org)
* on utilise les dépendances nodejs suivante [less](http://lesscss.org/) , [uglify-js](https://github.com/mishoo/UglifyJS2) , [uglifycss](https://github.com/fmarcia/UglifyCSS), et bien sur [bower](http://bower.io/).

  Less, uglify-js et uglifycss seront utilisé par les filtres assetic par la suite.
* à la fin de l'installation npm, on lance l'installation des packages front par **bower** : on utilise l'executable bower local du projet qui a été installé dans `node_modules/.bin/bower`

### Configurer bower

Créez tout d'abord un fichier `.bowerrc` afin de définir où les packages front vont être installé :

```json
{
    "directory": "web/bower_components"
}
```

Ici on configure bower pour installer les dépendances front dans le dossier `bower_components` dans le dossier `web` de symfony2.
On pourrait aussi les installer dans le dossier `Resources/public/bower_components` d'un bundle.


Ensuite il faut créer le fichier `bower.json` (éventuellement avec la commande `bower init` ou plutôt `node_modules/.bin/bower init` car on utlise une version de bower installé de façon locale par npm).

```json
{
  "name": "my-sf2-project",
  "authors": [
    "Martin Durand <martin.durand@example.com>"
  ],
  "private": true,
  "ignore": [
    "**/.*",
    "node_modules",
    "bower_components",
    "public/bower_components",
    "test",
    "tests"
  ],
  "dependencies": {
    "bootstrap": "v3.1.1",
    "jquery": "~2.1.1",
    "jquery-migrate": "~1.2.1",
    "font-awesome": "~4.1",
    "modernizr": "2.6.2"
  }
}
```

Ici, Bower est configuré pour rendre privé le projet symfony2 ( pas de publication accidentelle sur bower.io) et on utilise les dépendances suivantes : twitter bootstrap, jquery, font-awesome et modernizr.

### Configuration d'Assetic et Symfony2

Modification de la config d'assetic :

* `app/config/config.yml`

```yaml
assetic:
    debug: "%kernel.debug%"
    use_controller:
        enabled: false
    node: "/usr/local/bin/node" # chemin vers l'executable nodejs
    node_paths:
        # chemin vers les modules nodejs installés localement pour le projet par npm
        - "%kernel.root_dir%/../node_modules"  
        # chemin vers les modules nodejs du systeme, sert de fallback
        - "/usr/local/lib/node_modules"
    filters:
        less:
            apply_to: "\\.less$"
            # on utilise le lessc installé localement
            bin: "%kernel.root_dir%/../node_modules/.bin/lessc"
        cssrewrite:
            apply_to: "\\.css$"
```

* `app/config/config_prod.yml`

```yaml
assetic:
    filters:
        less:
            apply_to: "\\.less$"
            # on utilise le lessc installé localement
            bin: "%kernel.root_dir%/../node_modules/.bin/lessc"
        cssrewrite:
            apply_to: "\\.css$"
        uglifycss:
            # on utilise le uglifycss installé localement
            bin: "%kernel.root_dir%/../node_modules/.bin/uglifycss"
            apply_to: "\\.css$"
        uglifyjs2:
            # on utilise le uglifyjs installé localement
            bin: "%kernel.root_dir%/../node_modules/.bin/uglifyjs"
            apply_to: "\\.js$"
```

### Utilisation des assets dans twig

Desormais vous pouvez utiliser les balises assetic de twig pour compiler les fichiers less et rassembler et minifier tous les fichiers js et css

```
{% raw %}
{% stylesheets
   'bower_components/bootstrap/less/bootstrap.less'
    output='css/compiled/main.css'
%}
    <link href="{{ asset_url }}" type="text/css" rel="stylesheet" media="screen" />
{% endstylesheets %}

{% javascripts
        'bower_components/modernizr/modernizr.js'
        'bower_components/jquery/dist/jquery.js'
        'bower_components/jquery-migrate/jquery-migrate.js'
        'bower_components/bootstrap/js/affix.js'
        'bower_components/bootstrap/js/alert.js'
        'bower_components/bootstrap/js/button.js'
        'bower_components/bootstrap/js/carousel.js'
        'bower_components/bootstrap/js/collapse.js'
        'bower_components/bootstrap/js/dropdown.js'
        'bower_components/bootstrap/js/modal.js'
        'bower_components/bootstrap/js/tooltip.js'
        'bower_components/bootstrap/js/popover.js'
        'bower_components/bootstrap/js/scrollspy.js'
        'bower_components/bootstrap/js/tab.js'
        'bower_components/bootstrap/js/transition.js'
        output='js/compiled/main.js'
    %}
    <script type="text/javascript" src="{{ asset_url }}"></script>
{% endjavascripts %}
{% endraw %}
```

### On installe tout

Maintenant on peut tout installer :

```bash
php composer.phar install
php app/console assetic:dump --env=prod --no-debug
```

et voila !
