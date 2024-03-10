---
title: SASS & Eleventy
description: Comment intégrer SASS dans un site 11ty sans utiliser un gestionnaire de pipelines.
tags:
    - 11ty
    - sass
created: 2021-11-21
published: 2021-11-28
---

## Introduction

Durant la construction de ce blog, j'ai essayé plusieurs méthodes pour gérer les styles.
La majeure partie des articles que j'ai croisés impliquait l'utilisation d'un gestionnaire de pipelines comme [Webpack](https://webpack.js.org/), [gulp](https://gulpjs.com/) ou autre.
Parfois, certain-e-s utilisaient directement [SASS](https://sass-lang.com/guide) ou [PostCSS](https://postcss.org/) en CLI via des scripts NPM.  
Ces deux solutions me gênaient pour deux raisons :
- Concernant Webpack ou autre, il fallait installer une nouvelle dépendance au projet et cela n'allait pas trop avec l'approche minimaliste que je cherche à appliquer sur ce site.
- Pour les commandes CLI, cela impliquait de potentiellement avoir une duplication de la configuration (avoir deux fois le dossier `output` déclaré par exemple) et de lancer 11ty et PostCSS (ou SASS) en parallèle. Je ne trouvais pas cela élégant.

Après quelques recherches supplémentaires, je suis tombé sur [cet article](https://florian.ec/blog/cache-busting-eleventy-postcss/) qui explique comment gérer le cache navigateur pour les styles. Cet article, répondant à un problème que j'allais rencontrer plus tard, utilise une méthode basée sur l'utilisation de [template en JavaScript](https://www.11ty.dev/docs/languages/javascript/) proposée par 11ty pour gérer les styles.

L'ensemble du code présenté dans cet article est disponible [ici](https://gitlab.com/hex46/sass-eleventy).

## Quelques styles

Avant de commencer, nous allons créer un dossier `scss/` contenant les styles et quelques fichiers CSS :

```scss
// Fichiers _colors.scss (j'utilise les couleurs de la palette NordTheme ;))
$bg-color: #2e3440;
$text-color: #eceff4;
```

```scss
// Fichiers styles.scss
@import "colors";

html {
    height: 100%;
    width: 100%
}

html {
    background-color: $bg-color;
    color: $text-color;
}
```

Maintenant, passons à la partie JS.

## JavaScript Template

Ce type de template permet d'aller récupérer les données via des fonctions JavaScript et de générer une page à partir de celles-ci. On peut l'utiliser pour générer des pages à partir d'API Rest ou Graphql, mais il est aussi possible de s'en servir en y intégrant d'autres API comme celles dédiées au traitement de fichiers SCSS ou même JS.  
Il y a plusieurs manières de décrire un template JavaScript. Dans cet article, je vais me focaliser sur l'utilisation de `class` (eh oui, je suis dev Java à la base), mais vous pouvez le faire sous forme de `function` classiques ou encore de `Promise`.

Dans le dossier contenant les styles SCSS, nous allons déclarer un template JavaScript `scss.11ty.js` (l'extension `.11ty.js` est obligatoire) : 

``` js
// src: https://www.11ty.dev/docs/languages/javascript/#optional-data-method
class Test {
  // or `async data() {`
  // or `get data() {`
  data() {
    return {
      name: "Ted",
      // … other front matter keys
    };
  }

  render({name}) {
    // will always be "Ted"
    return `<p>${name}</p>`;
  }
}

module.exports = Test;
```
La fonction `data` est optionnelle, mais elle peut être utile pour déclarer l'URL cible de notre futur fichier CSS. Dans un autre contexte comme celui d'appels REST ou GraphQL, il peut être intéressant de récupérer les données dans cette fonction.  
La fonction `render` est la fonction principale de cette classe. Elle va nous permettre de générer notre page à partir de la chaine de caractères qu'elle retourne.
Cette seconde fonction prend en paramètre ce que `data` retourne.  

__Petit tips JS utilisé ci-dessus__ : il est possible de récupérer une ou plusieurs propriétés d'un objet passé en paramètre via ce que l'on appelle le [Destructuring assignment](https://developer.mozilla.org/fr/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment) de JavaScript. Dans l'exemple ci-dessus issu de la doc d'11ty, `render` récupère uniquement la variable `name`.  

Le template ci-dessus va générer le contenu HTML suivant :
```html
<p>ted</p>
```

## API SASS

Après avoir installé SASS (`npm i -D sass`), penchons-nous un peu sur son fonctionnement.

Comme on peut le voir sur la [documentation](https://sass-lang.com/documentation/js-api), il suffit d'indiquer un fichier en entrée puis de récupérer le résultat de la fonction `renderSync` :

```js
// src: https://sass-lang.com/documentation/js-api
const sass = require('sass'); // or require('node-sass');

const result = sass.renderSync({file: "style.scss"});
console.log(result.css.toString());
```

Dans cet exemple, la version synchrone de `render` est utilisée, mais il est tout à fait possible d'appeler une version asynchrone : 

``` js
// src: https://sass-lang.com/documentation/js-api
const sass = require('sass'); // or require('node-sass');

sass.render({
  file: "style.scss"
}, function(err, result) {
  if (err) {
    // ...
  } else {
    console.log(result.css.toString());
  }
});

```

La version asynchrone [peut être plus longue](https://sass-lang.com/documentation/js-api/modules#render) et est utile dans quelques cas que l'on ne rencontre pas ici. Pour cet article, je préfère donc rester sur l'utilisation de `renderSync`.  

Si on ajoute l'API SASS à notre JavaScript Template, on obtient quelque chose qui ressemble à ça :

``` js
const path = require('path');
const sass = require('sass');

/*
 * Il n'est pas nécessaire de garder le nom de la classe et
 * on peut directement l'exporter.
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
            file: rawFilepath
        });

        return sassRenderResult.css.toString();
    }
}
```

La fonction `data` déclarée ci-dessus permet de récupérer l'emplacement :
- du SCSS `styles.scss` ;
- du fichier CSS qui sera généré dans notre dossier de destination (`_site/css/styles.css`).

La méthode `render` est ensuite utilisée pour appeler l'API SASS et générer le contenu de notre fichier CSS via avec `sass.renderSync`. Il suffira ensuite de récupérer la chaîne de caractères générée via `sassRenderResult.css.toString()`.  

Le contenu du fichier `_site/css/styles.css` devrait ressembler à ceci :

```css
html {
  height: 100%;
  width: 100%;
}

html {
  background-color: #2e3440;
  color: #eceff4;
  display: flex;
}
```

L'API SASS propose des options supplémentaires, notamment la possibilité de minifier le CSS généré : 

```js
    ...

    render({ rawFilepath }) {
        var sassRenderResult = sass.renderSync({
            file: rawFilepath,
            outputStyle: "compressed"
        });

        return sassRenderResult.css.toString();
    }

    ...
```

Avec le paramètre et de la valeur `outputStyle: "compressed"`, on obtiendra le fichier CSS suivant : 

```css
html{height:100%;width:100%}html{background-color:#2e3440;color:#eceff4;display:flex}
```

Il existe deux types d'options différentes : 
- Les [LegacyStringOptions](https://sass-lang.com/documentation/js-api/interfaces/LegacyStringOptions) - utilisées ici pour minifier le CSS.
- Les [LegacyFileOptions](https://sass-lang.com/documentation/js-api/interfaces/LegacyFileOptions) - utilisées pour la gestion des fichiers et dossiers.

Pour plus d'information concernant `render`, `renderSync` et de leurs paramètres, je vous invite à vous plonger dans la documentation 😉.

## Fin ?

Pour des projets simples, ce type de template devrait suffire. Cependant, dès que l'on souhaite utiliser des règles CSS particulières (comme `flex`), il sera préférable d'intégrer un outil du type [autoprefixer](https://github.com/postcss/autoprefixer). De plus, le problème du cache navigateur n'est ici pas traité. Ainsi, à chaque livraison de `styles.css`, il faudra vider le cache de chaque navigateur qui ont récemment consulté le site web...

On verra dans un prochain article comment prendre en compte ces deux problématiques.