---
title: "Normal Form Designators"
date: 2023-04-12T15:31:47+08:00
draft: false

categories: []
tags: ['Logiciel', 'Programmation']
toc: false
author: "bllaude"
---
[Brian Cantwell Smith](https://en.wikipedia.org/wiki/Brian_Cantwell_Smith) fait partie de ces nombreuses personnes qui sont bien plus intelligentes que moi. C’est grâce à sa thèse au MIT, intitulée [« Procedural Reflection in Programming Languages »](https://publications.csail.mit.edu/lcs/pubs/pdf/MIT-LCS-TR-272.pdf) — un sujet à part entière —, que j’ai découvert son travail pour la première fois.

<!--more-->

C’est dans cet article que j’ai appris les concepts présentés dans ce billet. Je ne sais donc pas vraiment si je dois attribuer ces concepts à Brian, mais je le ferai, car je ne sais pas à qui d’autre les attribuer.

L'acquisition de connaissances est un effort collectif, et j'adresse toute ma gratitude à tous ceux qui y ont contribué.

> **Avertissement:** Il se peut que j'utilise ici des termes de manière incorrecte. Si tel est le cas, merci de remplacer, dans toutes les occurrences des expressions figurant dans le livre de Brian, par des mots inventés qui traduisent fidèlement le sens que j'ai donné ici.

`*pripl` traite du développement de LISP-3, un langage Scheme doté de capacités réflexives. Le sujet est trop vaste pour que je puisse l'aborder en un seul article, mais soyez assurés que je le ferai à l'avenir.

D'un autre côté, on a souvent proposé d'utiliser les « Normal Form Designators », et j'ai trouvé que c'était une excellente idée.

### Fundamentals
À la base, la NFD consiste à regrouper sémantiquement le résultat de l'évaluation d'une instruction, quel que soit le degré d'avancement de cette évaluation.

À ce titre, les instructions suivantes du schéma se rapportent toutes au même Normal Form Designator :
```lisp
(if true
    (* 23 (+ 4 2))
    0)

(* 23 (+ 4 2))
(* 23 6)
69
```

Notez que ce qui suit fait partie du même ensemble **NFD** que ce qui suit ; il n'est donc pas nécessaire d'inclure toutes les étapes d'une même évaluation :
`(- 70 1)`

Cette idée me plaît pour plusieurs raisons, mais si j’écris cet article, c’est surtout pour en profiter afin d’exprimer mes frustrations concernant les logiciels.

À mon humble avis, tous ces soi-disant « progrès », notamment en ce qui concerne le Web moderne, ne parviennent pas réellement à améliorer les groupes NFD que la technologie prétend améliorer.

Nous utilisons toujours Internet pour effectuer quelques tâches élémentaires :
- Nous envoyer des messages
- Partager des fichiers
- Consulter des contenus multimédias

Et au lieu de proposer des méthodes efficaces et adaptées au contexte pour y parvenir, nous ne cessons de créer des expressions de plus en plus complexes qui reviennent au même, en les qualifiant de *« progrès »* parce qu’elles sont plus difficiles à comprendre pour le grand public.

Nous continuons d’affirmer que l’*« operating system improvement »* consiste à créer sans cesse une couche d’abstraction supplémentaire par-dessus les mêmes signifiants dénués de sens concernant ce que les utilisateurs sont en droit d’attendre d’un système d’exploitation, ou à les réinventer.

Ou alors, plutôt que de créer quelque chose de différent, on continue simplement à construire.

### Fin
J'en ai tout simplement marre de voir que les technologies les plus récentes ne sont que des trucs incroyablement lourds et inutiles qui nécessitent l'intervention d'une grande entreprise et des ordinateurs toujours plus puissants pour mettre en œuvre des fonctionnalités qui existent depuis des lustres.

J'en ai marre de voir des gens formés à ce que l'industrie des outils a décidé d'appeler *« la mode du mois »* et qui n'essaient jamais rien de vraiment unique.

Je souhaite trouver de nouveaux « Normal Form Designators » et faire de leur caractère distinctif un attribut positif en soi.

`C-c C-x`
