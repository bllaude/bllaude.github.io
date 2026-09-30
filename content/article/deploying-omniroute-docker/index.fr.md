---
title: "Déploiement d'OmniRoute sur Docker"
date: 2026-08-29T16:32:11+08:00
draft: false

categories: []
tags: ['IA', 'Docker', 'Compose', 'Agentic']
toc: false
author: "bllaude"
---
Pour moi, exécuter Omniroute sur Docker est la meilleure solution par défaut, tout simplement parce que je ne veux pas qu'une installation globale de npm vienne polluer l'environnement de mon nœud hôte, et que cela s'intègre parfaitement, que la machine soit un serveur domestique, un VPS ou ma machine virtuelle jetable.

<!--more-->

### Choisir la bonne image cible et compose profile
Le projet fournit un fichier Dockerfile en plusieurs étapes comprenant 3 cibles de compilation. `runner-base` correspond au runtime autonome Next.js de production, sans CLI de fournisseur intégrée ; c’est le choix approprié si Omniroute doit uniquement servir de proxy pour les requêtes provenant d’une instance Codex exécutée ailleurs.

`runner-cli` étend ces fonctionnalités avec git, docker.io, docker-compose et les global install de `@openai/codex`, `@anthropic-ai/claude-code`, `droid` et `openclaw` — ce qui est utile pour les workflows où l’agent lui-même doit résider à l’intérieur du conteneur, mais constitue une solution excessive pour le cas de figure courant où *« l’agent s’exécute sur mon ordinateur portable et OmniRoute s’exécute sur un serveur ».*

Compose les présente sous forme de profils plutôt que de vous obliger à jongler manuellement avec les cibles du Dockerfile : « base » correspond à l’image « runner-base » minimale, `cli` intègre les CLI fournies, et « host » est un profil orienté Linux qui monte en lecture seule les binaires CLI de l’hôte et les répertoires de configuration. Un quatrième profil, `cliproxyapi`, exécute le sidecar CLIProxyAPI sur le port 8317 pour la mise en proxy des CLI en amont et peut être combiné avec n’importe lequel des autres, par exemple : `docker compose --profile cli --profile cliproxyapi up -d`. Pour une configuration simple, « base » est presque toujours le bon point de départ :

```bash
docker compose --profile base up -d
```

Si vous préférez vous passer complètement de Compose, l'exécution directe via `docker run` est tout aussi valable et c'est ce que la plupart des déploiements sur un seul serveur finissent par utiliser une fois que le dimensionnement de la mémoire (voir ci-dessous) est bien réglé :

```bash
docker run -d --name omniroute --restart unless-stopped --stop-timeout 40 \
  -p 127.0.0.1:20128:20128 -v omniroute-data:/app/data \
  diegosouzapw/omniroute:latest
```

Il est recommandé de configurer par défaut la liaison vers `127.0.0.1` plutôt que vers `0.0.0.0`, et de ne l'ouvrir qu'une fois que vous aurez déterminé comment vous souhaitez réellement exposer le tableau de bord et l'API : via Cloudflare Tunnel, Tailscale ou un proxy inverse avec TLS en amont, toutes ces options étant documentées séparément dans la documentation du projet.

Comme l’image Docker définit toujours explicitement `OMNIROUTE_MEMORY_MB`, le mécanisme de repli calibré en fonction de la RAM propre au lanceur bare-metal (environ 35 % de la RAM de l’hôte, plafonné entre 512 Mo et 4 Go) ne se déclenche jamais sous Docker - vous êtes censé définir vous-même cette valeur pour les déploiements en conteneur, ce qui diffère du comportement d’`omniroute serve` en dehors d’un conteneur et mérite d’être pris en compte avant de rechercher un comportement d’autoscaling qui n’existe pas.