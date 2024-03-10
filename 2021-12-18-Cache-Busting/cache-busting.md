---
title: Cache busting & 11ty
description: Aprés avoir utilisé SASS, PostCSS et autoprefixer, on va voir comment gérer le cache busting.
tags:
    - 11ty
    - css
    - cache
create: 2021-12-18
published: 2022-01-06
---

## Introduction

Voici la suite des deux précédents articles concernant l'intégration de SASS et PostCSS dans 11ty.
Je vous invite à les lire si ce n'est pas déjà fait ou si vous n'êtes pas familier avec l'utilisation des templates JS.  

## La gestion du cache navigateur

Le [cache navigateur](https://fr.wikipedia.org/wiki/Cache_web) permet, entre autres, de stocker les éléments d'une page afin de l'afficher plus rapidement. Il permet aussi aux petites connexions d'éviter la récupération de données déjà effectuée.  

Si je regarde l'onglet "Réseau" de Firefox pour ce site web, j'obtiens ceci :

{% image "./src/pages/posts/2021-12-18-Cache-Busting/cache-firefox.png", "Image de l'onglet 'Réseau' de Firefox", "center", " " %}

On peut voir que l'ensemble des éléments sont récupérés du cache du navigateur.
Cependant, lors de la livraison d'une nouvelle version d'un site, il est souvent nécessaire de forcer la suppression de celui-ci afin d'avoir les dernières modifications livrées. Il existe plusieurs méthodes pour gérer cela. Dans la suite de cet article, nous allons voir la technique du cache busting en l'intégrant directement dans notre template.

Une méthode simple de cache busting est de changer le nom du ou des fichiers CSS pour forcer le téléchargement de ces derniers.  
Ce nom peut être généré via un [hash](https://fr.wikipedia.org/wiki/Fonction_de_hachage) dudit fichier à partir de son contenu. L'avantage de cette méthode est que le nom change uniquement si le contenu change.
Donc, si on livre une version du site sans modification des styles, les postes clients n'auront pas à télécharger une nouvelle fois ces derniers.  

Pour cela, on peut utiliser l'API `crypto` de NodeJS et qui nous permet d'utiliser des algorithmes de chiffrement pour la génération du hash.
Voici un exemple de fonction utilisant l'algorithme [MD5](https://fr.wikipedia.org/wiki/MD5) :

```js
/**
 * src: 
 * - https://florian.ec/blog/cache-busting-eleventy-postcss/
 * - https://melvingeorge.me/blog/create-md5-hash-nodejs#full-solution
 */
const crypto = require('crypto');

// Le paramètre rawFile va contenir notre fichier à analyser
// (fichier contenant nos styles dans notre exemple).  
module.exports = function generateHash(rawfile) {
    checkParameters(rawfile);
    return getHash(rawfile);
}

function checkParameters(rawfile) {
    if (!rawfile)
        throw new Error('Error: some parameters are empty');

    if(rawfile.length === 0 )
        throw new Error('Error: some parameters are blank');
}

function getHash(rawfile) {
    const hmacResult = crypto.createHmac('md5', ''); // secret si besoin en second paramètre
    const hash = hmacResult.update(rawfile);
    return hash.digest('hex');
}
```
Le code ci-dessus est placé dans un fichier `libs/generate-hash.js` d'où la présence d'un `module.exports = ...`.
Cependant, il est tout a fait possible de mettre ces fonctions directement dans le template JS.  

Voici ce que ça donne lorsqu'on l'intègre : 

```js
const generateHash = require('../libs/generate-hash');
...
    data() {
        const scssDir = path.join(__dirname, '.');
        const rawFilepath = path.join(scssDir, 'styles.scss');

        const sassRenderResult = sass.renderSync({
            file: rawFilepath,
            outputStyle: "compressed",
        });

        const rawCss = sassRenderResult.css.toString();
        const hash = generateHash(rawCss); // Génération du hash

        return {
            permalink: `/css/styles.${hash}.css`, // Ajout du hash dans le nom du fichier
            rawFilepath: rawFilepath,
            rawCss: rawCss
        }
    }
...
```

On génère le hash à partir du résultat de `sass.renderSync` puis on indique à 11ty que le fichier généré sera créé ici : `/css/styles.${hash}.css`.
`${hash}` faisant référence à la variable contenant notre hash. J'utilise ici un [Template strings](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals) afin d'éviter les concaténations comme `"/css/styles." + {hash} " + ".css"` (note : les string sont immuables en JS).

Avec cette méthode, on obtient un nom de fichier qui ressemble à ceci : `styles.940b2e0d171b2a9077a888f9f55e8c25.css`.  
Si on change une règle (par exemple, on remplace `$bg-color: #2e3440;` par `$bg-color: red;`), le nom du fichier change est modifié : `styles.d6d3a3d4f557db17b4f330521bb1dc47.css`.  

### Récupération du lien dans notre head

C'est terminé ? Pas encore. Il nous reste à intégrer le fichier généré dans le `head` du site.  
La méthode la plus simple est de récupérer le résultat de `data()` afin de récupérer `permalink`. Une manière d'implémenter cette méthode est de créer un [JavaScript data file](https://www.11ty.dev/docs/data-js/) (par exemple, `css.js`) dans le dossier `_data` d'11ty :

```js
// Fichier _data/css.js
const SCSSBuild = require('../scss/scss.11ty');

module.exports = async function() {
    const scssBuildInstance = new SCSSBuild();
    const data = scssBuildInstance.data();

    return data;
}
```

Avec cela, on va pouvoir récupérer le chemin du fichier et l'ajouter dans le `head` :

```html
{% raw %}<link rel="stylesheet" type="text/css" href="{{ css.permalink }}"> {% endraw %}
```

## Limites et autres solutions

Attention, cette méthode a un inconvénient : la fonction de hash est appélée au moins deux fois :
une fois pour le traitement du CSS et une autre pour la récupération du hash dans la balise `<link>`.  
Pour mon site, il arrive qu'11ty indique que le traitement du fichier `css.js` est trop long :

{% image "./src/pages/posts/2021-12-18-Cache-Busting/benchmark-css.png", "Traitement trop long pour CSS", "center", " " %}

9.8% du temps de la génération du site, c'est beaucoup trop important pour récupérer une chaîne de caractère 😉.
Pour mon cas, ce n'est pas trop un problème : je n'ai qu'un fichier CSS créé et le temps de traitement reste faible (0.62s pour le site avec l'option `--serve` à l'heure où j'écris ces lignes).  

Voici quelques pistes pour améliorer le traitement des styles :
- Modifier `scss.11ty.js` afin d'effectuer le traitement de `data()` une seule fois (via un cache par exemple)
- Déplacer le hash dans les paramètres HTTP de `<link>` en le supprimant du nom du fichier
    - par exemple : `{% raw %}<link rel="stylesheet" type="text/css" href="css/styles.css?v={{ css.hash }}"> {% endraw %}`
- Mettre les styles en inline (appelé aussi "[Internal CSS](https://www.w3schools.com/css/css_howto.asp)")
    - Attention, cette méthode a aussi ses limites comme l'explique Jecelyn Yeen dans son article ["Minifying HTML, JavaScript, CSS - Automate Inline"](https://jec.fyi/blog/minifying-html-js-css)

Voilà, vous savez comment intégrer SASS, PostCSS et gérer le cache busting directement via les JavaScript Template d'11ty 😉.