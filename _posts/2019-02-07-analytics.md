---
layout: post
title:  "Analytics avec Clicky sur Jekyll"
date:   2019-02-07 19:33:12 +0100
categories: other
---

## Les alternatives à Google Analytics

L'autre jour je voulais mettre en place de l'analytics sur ce blog, et je ne voulais pas utiliser
Google Analytics. J'ai tapé "alternative to google analytics" sur DuckDuckGo et j'ai cliqué sur la première page :

https://www.searchenginejournal.com/9-google-analytics-alternatives/92071/

Mes critères étaient les suivants :
- complètement gratuit
- ne nécessite pas d'héberger quelque chose sur son serveur
- fonctionnalités basiques

Dans la liste, Piwik a retenu mon attention car j'en avais déjà entendu parlé. 
C'est un projet open-source, mais il faut soit l'héberger soit-même soit utiliser le service cloud payant.

Le seul dans la liste qui répondait à tous mes critères c'est Clicky.


## Démarrer avec Clicky

La procédure d'inscription sur Clicky est rapide est directe.

Je me crée donc un compte.
Sur la page suivante on me demande l'adresse du site.
La page suivante invite à vérifier quelques paramètres,
puis on arrive sur la page qui donne le code de tracking à insérer dans son site.
Une case à cocher permet d'intégrer une image invisible pour suivre les visiteurs sans Javascript.
Je la coche parce que pourquoi pas ?


## Inclusion du code de tracking dans Jekyll

Du fait de l'image dans le code de tracking, celui-ci doit être inclu dans la balise body de la page.

Jekyll utilise un système de thème pour contrôler comment sont rendues les pages du site.
Le thème par défaut que j'utilise est minima.

La commande `bundle show minima` indique où trouver les fichiers du thème.
Chez moi c'était dans `/Library/Ruby/Gems/2.3.0/gems/minima-2.5.0`.

Dans le projet Jekyll il est possible de surcharger n'importe quel fichier du thème
en créant un fichier du même nom.
Après avoir exploré les fichiers du thème j'identifie celui que je dois modifier : `_layouts/default.html`.
Je copie colle le fichier dans mon projet et ajoute le code de tracking.
Voilà le fichier modifié :

```html
{% raw %}
<!DOCTYPE html>
<html lang="{{ page.lang | default: site.lang | default: "en" }}">

  {%- include head.html -%}

  <body>

    {%- include header.html -%}

    <main class="page-content" aria-label="Content">
      <div class="wrapper">
        {{ content }}
      </div>
    </main>

    {%- include footer.html -%}

    {%- if jekyll.environment == 'production' -%}
      <script>var clicky_site_ids = clicky_site_ids || []; clicky_site_ids.push(123456789);</script>
      <script async src="//static.getclicky.com/js"></script>
      <noscript><p><img alt="Clicky" width="1" height="1" src="//in.getclicky.com/123456789ns.gif" /></p></noscript>
    {%- endif -%}

  </body>

</html>
{% endraw %}
```

Je rajoute `{% raw %}{%- if jekyll.environment == 'production' -%}{% endraw %}` pour désactiver le tracking en dev,
même s'il semble que Clicky n'enregistre pas les visites depuis les domaines locaux.
En effet, j'ai fait plusieurs tests en local sans constater aucun changement dans les stats,
alors qu'une fois en prod un seul refresh a suffit pour voir apparaitre une visite.


Inspiration:
[https://desiredpersona.com/google-analytics-jekyll/](https://desiredpersona.com/google-analytics-jekyll/)