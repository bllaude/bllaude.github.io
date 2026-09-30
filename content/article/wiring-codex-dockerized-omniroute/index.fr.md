---
title: "Codex de câblage sur Omniroute en Docker"
date: 2026-08-31T17:01:53+08:00
draft: false

categories: []
tags: ['Codex', 'Docker', 'Omniroute']
toc: false
author: "bllaude"
---
Pour tous ceux qui utilisent Codex avec plusieurs backends (le point de terminaison officiel d’OpenAI, Kimi ou DeepSeek, selon ce qui s’avère le moins cher ou le plus rapide cette semaine-là), le fait de placer Omniroute en amont permet à Codex de n’avoir à connaître qu’une seule URL de base, tandis qu’Omniroute gère les interruptions de service des fournisseurs, les rate-limit et la rotation des identifiants de l’autre côté.

<!--more-->

## Memory Sizing
Je me suis heurté à ce problème la première fois que j’ai configuré Codex pour qu’il utilise un conteneur Omniroute auto-hébergé. L’image est fournie avec `OMNIROUTE_MEMORY_MB=1024`, ce qui définit une limite maximale de 1 GiB pour l’« old space » de V8 via `NODE_OPTIONS=--max-old-space-size`. Cela suffit pour le tableau de bord et un trafic de chat léger, mais il s’agit d’un seuil minimum, et non d’une taille adaptée à la production. Les appels `POST /v1/responses` de Codex transportent de longs historiques de messages et des définitions d’outils, et le pipeline de compression `RTK+Caveman` d’Omniroute conserve plusieurs graphes en mémoire pendant leur traitement.

Une session unique coding-agent nécessite généralement `OMNIROUTE_MEMORY_MB=8192`, la limite de mémoire du `cgroup` du conteneur devant être fixée bien au-dessus de cette valeur — `--memory=10g` ou plus — car les *buffer* natifs, SQLite et les données intermédiaires de compression se trouvent tous en dehors du tas V8 et ne sont pas couverts par la limite maximale du tas elle-même. On a observé que deux requêtes à long contexte se chevauchant pouvaient provoquer l’abandon pur et simple d’un tas de 12 Gio avec le message « `FATAL ERROR: Reached heap limit` » ; par conséquent, si vous exécutez Codex en parallèle avec d’autres processus, augmentez davantage la taille du tas ou limitez la concurrence à une seule requête lourde en cours d’exécution par processus, plutôt que d’augmenter la valeur de `OMNIROUTE_CHAT_MAX_HEAVY_IN_FLIGHT` en espérant que la mémoire vive suive.

La commande corrigée se présente comme suit :
```bash
docker run -d --name omniroute --restart unless-stopped --stop-timeout 40 \
-e OMNIROUTE_MEMORY_MB=8192 --memory=10g \
-p 127.0.0.1:20128:20128 -v omniroute-data:/app/data \
diegosouzapw/omniroute:latest
```

C’est là que je bute d’un point de vue conceptuel : la commande `setup-codex` d’Omniroute écrit des fichiers réels dans ~`/.codex` sur la machine qui l’exécute, et si cette machine est le conteneur lui-même, l’écriture se fait dans le répertoire personnel éphémère du conteneur (`/home/node`, puisque l’image s’exécute sous l’utilisateur « node » sans privilèges) et disparaît dès que le conteneur est recréé. Omniroute détecte effectivement ce problème et le signale, plutôt que d’afficher silencieusement le message « success: the CLI exits with status 2 » ; la couche API renvoie alors un code 422 avec `containerEphemeralTarget: true`.

Le modèle documenté et recommandé sépare ces deux aspects : le conteneur fournit l’API compatible avec OpenAI, tandis que l’interface en ligne de commande (CLI) — installée sur votre ordinateur portable ou workspace, là où Codex s’exécute — se charge de créer le fichier de configuration que Codex lit :

```bash
docker compose --profile base up -d

npm install -g omniroute
omniroute connect http://localhost:20128
omniroute setup-codex
```

Omniroute connect redirige l’interface CLI installée localement vers le port exposé du conteneur, puis la commande `omniroute setup-codex` crée le fichier `~/.codex/*.config.toml` sur l’hôte, en configurant Codex pour qu’il utilise l’URL de base d’Omniroute et en y injectant la clé API générée par le tableau de bord sous `Dashboard → Endpoints`. C'est la configuration idéale pour le cas de figure extrêmement courant où Codex s'exécute localement et Omniroute sur un serveur ou dans un conteneur local.

Si vous souhaitez spécifiquement que le conteneur puisse écrire directement dans les fichiers de configuration de l’hôte — par exemple, pour orchestrer la configuration depuis un script s’exécutant déjà au sein du conteneur —, le profil `Compose` de l’hôte effectue un montage lié des répertoires concernés et définit `CLI_CONFIG_HOME` sur la racine du montage :

```yaml
environment:
  - CLI_CONFIG_HOME=/host-home
  - CLI_ALLOW_CONFIG_WRITES=true
volumes:
  - ~/.codex:/host-home/.codex:rw
  - ~/.claude:/host-home/.claude:rw
```
Omniroute considère la présence d'un montage « bind » valide, vérifiée en lisant `/proc/self/mountinfo`, comme le signe qu'un chemin est suffisamment fiable pour y écrire ; il refuse toute écriture sur tout ce qui n'est pas réellement monté depuis l'hôte, ce qui empêche le « footgun » d'écriture éphémère de se produire, même si vous essayez.

Il existe également une solution de secours prévue pour les cas où Codex est censé résider à l’intérieur même du conteneur (le profil CLI) : en passant l’option `--allow-container-write` à `setup-codex`, ou en définissant `OMNIROUTE_ALLOW_CONTAINER_CONFIG_WRITE=true` sur le serveur, l’écriture est autorisée, accompagnée d’un avertissement explicite indiquant qu’elle ne sera pas conservée lors de la recréation du conteneur.

## Redis, la persistance et les problèmes qui surgissent au redémarrage
Redis prend en charge le *distributed rate limiter* et le *shared cache* d’Omniroute, et le service Redis dans `docker-compose.yml` ne dispose pas de profil de contrôle d’accès : il démarre quel que soit le profil que vous choisissez. Par défaut, il n’est accessible qu’à l’adresse `127.0.0.1`, car il s’exécute sans `requirepass` ; si vous avez besoin qu’il soit accessible depuis d’autres points du réseau, définissez `REDIS_BIND_HOST=0.0.0.0` et ajoutez `--requirepass` à la commande du service lors de cette même modification, et non par la suite.

La désactivation de Redis est possible mais déconseillée, car le rate-limiter se rabat alors sur une solution de secours en mémoire qui ne survit pas aux redémarrages et ne s'étend pas à l'ensemble des processus.

Deux détails opérationnels sont faciles à négliger et revêtent tous deux une importance particulière pour un workflow Codex en cours de session lorsqu'un problème survient. Tout d’abord, omniroute utilise SQLite en mode WAL par défaut, et il a besoin que la commande `docker stop` s’exécute jusqu’au bout — et non qu’elle soit interrompue — afin de pouvoir enregistrer un point de contrôle dans `storage.sqlite` ; les fichiers Compose fournis définissent un délai de grâce de 40 secondes pour l’arrêt de cette raison, et si vous utilisez la commande `docker run` seule, conservez l’option `--stop-timeout 40` plutôt que de vous fier à la valeur par défaut de Docker.

Deuxièmement, le déploiement par défaut consiste en un processus Node soutenu par un seul module d’écriture SQLite, un point c’est tout : il n’existe aucun moyen pris en charge d’exécuter plusieurs répliques sur le même fichier SQLite, et le faire corrompt la base de données. Une recréation, un redémarrage ou un contrôle d’intégrité échoué entraîne une interruption totale de toutes les sessions Codex en cours : les flux SSE sont interrompus, et les requêtes qui arrivent pendant cette interruption reçoivent simplement un code d’erreur `502 Bad Gateway` de la part de la couche en amont, plutôt que l’erreur JSON que produirait Omniroute lui-même, ce qui peut prêter à confusion en donnant l’impression d’une défaillance côté fournisseur plutôt que d’une défaillance de l’infrastructure. Si vous avez réellement besoin de plus d’une ou deux sessions Codex à long contexte simultanées, la procédure documentée consiste à utiliser `N` processus indépendants avec des volumes `DATA_DIR` distincts et `QUOTA_STORE_DRIVER=redis` pour le comptage partagé des quotas — et non pas `replicas > 1` sur un stockage partagé.

Veillez à toujours monter `/app/data` sur un volume nommé, quel que soit le profil choisi ; c'est là que se trouvent la db, les identifiants cryptés des fournisseurs et la configuration. Si vous omettez cette étape, chaque recréation de conteneur repartira de zéro.

## Une base de référence raisonnable
En rassemblant tous les éléments, un déploiement « single-box » destiné à servir de front-end à Codex se présente de manière fiable comme suit dans Compose :

```yaml
services:
  omniroute:
    image: diegosouzapw/omniroute:latest
    container_name: omniroute
    restart: unless-stopped
    stop_grace_period: 40s
    environment:
      OMNIROUTE_MEMORY_MB: "8192"
      OMNIROUTE_WS_BRIDGE_SECRET: "<generate a strong random value>"
    volumes:
      - omniroute-data:/app/data
    ports:
      - "127.0.0.1:20128:20128"
    deploy:
      resources:
        limits:
          memory: 10g

volumes:
  omniroute-data:
```

Lancez la commande `docker compose --profile base up -d`, puis exécutez `npm install -g omniroute`, `omniroute connect http://localhost:20128` et `omniroute setup-codex` depuis la machine sur laquelle Codex s'exécute réellement.

Fixez le tag de l'image à une version spécifique `X. Y.Z` spécifique plutôt que de suivre `:latest` s’il ne s’agit pas d’une machine personnelle que vous pouvez recréer à votre guise — `:latest` ne change que lorsqu’une `version SemVer` stable est publiée et promue ; ce n’est donc pas une garantie d’actualité pour ce qui a été intégré hier dans la branche principale. Le verrouillage est ce qui rend le déploiement reproductible lorsque vous devrez inévitablement revenir en arrière après un problème de fournisseur ou une mauvaise surprise liée au dimensionnement de la mémoire.