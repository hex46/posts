---
title: PostCSS & 11ty
description: PostCSS dans un site 11ty avec les templates JS.
tags:
    - 11ty
    - sass
    - postcss
created: 2021-12-05
published: 2021-12-19
---

## Introduction

Dans l'article précédent, nous avons vu comment intégrer SASS dans 11ty en passant par les templates JavaScript. La méthode présentée ne permet pas de gérer certaines règles CSS. Par exemple, `display:flex` doit être remplacée par `display:-webkit-box` ou `display:-ms-flexbox` pour certaines versions de navigateurs. L'API SASS ne nous donne pas les outils pour le faire, mais PostCSS dispose du plugin `autoprefixer` qui réalise cette tâche.
Nous allons donc voir comment les intégrer à 11ty.  

L'ensemble du code présenté dans cet article est disponible [ici](https://gitlab.com/hex46/postcss-sass-eleventy).

## Quelques éléments de code

Avant de commencer, voici un petit rappel des différents éléments que nous avions dans le dernier article.  

Fichier `_colors.scss` :
```sass
$bg-color: #2e3440;
$text-color: #eceff4;
```

Fichier `styles.scss` :
```sass
@import "colors";

html {
    height: 100%;
    width: 100%
}

html {
    background-color: $bg-color;
    color: $text-color;
    display: flex; // on verra plus tard pourquoi
}
```

Fichier `scss.11ty.js` :
```js
const path = require('path');
const sass = require('sass');

/*
 * Il n'est pas nécessaire de garder le nom de la classe.
 * On peut directement l'exporter.
 */
module.exports = class {

    data() {
        const scssDir = path.join(__dirname, '.');
        const rawFilepath = path.join(scssDir, 'styles.scss');

        return {
            permalink: '/css/styles.css',
            rawFilepath: rawFilepath
        }
    }

    render({ rawFilepath }) {
        var sassRenderResult = sass.renderSync({
            file: rawFilepath,
            outputStyle: "compressed",
        });

        return sassRenderResult.css.toString();
    }
}
```

## Intégration de PostCSS

Comme [SASS](https://sass-lang.com/documentation/js-api), [PostCSS](https://postcss.org/api/) possède une API JavaScript. 
La fonction principale de cette API est `postcss()` et prend en paramètre un tableau de plugins :

```js
postcss(plugins)
    .process(css, { from, to })
    .then(result => {
        console.log(result.css)
});
```

Comme le montre l'exemple ci-dessus, il est possible d'enchaîner les appels de fonction à partir du premier appel de `postcss()`. C'est ce qu'on appelle le [methode chaining](https://en.wikipedia.org/wiki/Method_chaining). `process()` s'occupe de transformer le CSS et retourne une `Promise` qui est ensuite gérée par `then()` avec le résultat de la transformation.

Pour intégrer PostCSS dans notre projet, il suffit de l'installer puis de l'intégrer dans notre template JS :

```js
const path = require('path');
const sass = require('sass');
const postcss = require('postcss');

module.exports = class {

    data() {
        const scssDir = path.join(__dirname, '.');
        const rawFilepath = path.join(scssDir, 'styles.scss');

        const sassRenderResult = sass.renderSync({
            file: rawFilepath,
            outputStyle: "compressed",
        });

        const rawCss = sassRenderResult.css.toString();

        return {
            permalink: '/css/styles.css',
            rawFilepath: rawFilepath,
            rawCss: rawCss
        }
    }

    render({ rawCss }) {
        // ici
        return postcss()
            .process(rawCss)
            .then((result) => result.css);
    }
}
```

Dans ce code, j'ai fait le choix de déplacer `sass.renderSync` dans la méthode `data` parce que je considère le CSS produit par cette fonction comme une donnée à fournir lors de la génération de ma page. Mais il est tout à fait possible de laisser `sass.renderSync` dans la méthode `render`.  

Pour ajouter autoprefixer, il suffit de l'installer puis de l'ajouter dans l'appel de la fonction :

```js
...
    render({ rawCss }) {
        return postcss([require('autoprefixer')]) // ici
            .process(rawCss)
            .then((result) => result.css);
    }
...
```

Ce qui nous donne le résultat suivant :

```css
html{height:100%;width:100%}html{background-color:#2e3440;color:#eceff4;display:-webkit-box;display:-ms-flexbox;display:flex}
```

On y retrouve bien les règles `display:-webkit-box` et `display:-ms-flexbox` qui ont été ajoutées par le plugin.  
Pour indiquer à celui-ci la liste des versions de navigateurs à prendre en charge, il faut configurer l'une des dépendances d'autoprefixer : [browserslist](https://github.com/browserslist/browserslist#queries).  

browserslist est un outil JavaScript qui permet d'indiquer les navigateurs ciblés pour un projet. Il y a plusieurs méthodes pour le configurer.  
Par exemple, il est possible de l'ajouter dans le `package.json` :

```json
...
    "browserslist": "last 5 version"
...
```

Ou bien en paramètre d'autoprefixer :

```js
...
    postcss([require('autoprefixer')({ overrideBrowserslist: "last 5 version" })])
...
```

On peut ajouter d'autres plugins comme [cssnano](https://cssnano.co/docs/introduction#what-is-cssnano) pour remplacer la ligne `outputStyle: "compressed"`  ou ajouter la syntaxe [scss](https://github.com/postcss/postcss-scss) directement dans la configuration de PostCSS et ainsi éviter l'utilisation de l'API SASS dans le template.  

Maintenant, vous savez comment utiliser PostCSS dans un template JS d'11ty !  
Dans l'article suivant, nous allons voir comment gérer le cache du fichier de styles.