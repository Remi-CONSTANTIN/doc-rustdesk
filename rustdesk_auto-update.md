# Introduction
Ce guide rapide porte sur la mise en place de la mise à jour automatique de Rustdesk.  
En effet, la sécurité d'une instance passe aussi et surtout par la correction de potentielles failles.  

Nous allons ici utiliser la solution `What's Up Docker` étant en quelque sorte l'héritier de `Watchtower`.

# Mise en pratique

1. Commencez par préparer votre instance Rustdesk en ajoutant le tag `"wud.watch.digest=true"` sur ses conteneurs `hbbs` et `hbbr`
```
nano /opt/rustdesk/docker-compose.yml
```

Cela devrait ressembler à ça :
```
services:
  hbbs:
    container_name: hbbs
    image: rustdesk/rustdesk-server:latest
    command: hbbs -r <Votre-IP-Publique>:21117 -k _
    volumes:
      - ./data:/root
    network_mode: "host"
    depends_on:
      - hbbr
    restart: unless-stopped
    labels:
      - "wud.watch.digest=true"
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL

  hbbr:
    container_name: hbbr
    image: rustdesk/rustdesk-server:latest
    command: hbbr -k _
    volumes:
      - ./data:/root
    network_mode: "host"
    restart: unless-stopped
    labels:
      - "wud.watch.digest=true"    
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL

```

2. Chose faite, créez un répertoire dédié à` What's Up Docker` dans `/opt`
```
mkdir /opt/WhatsUpDocker
```

3. Rendez-vous sur [htpasswd.utils.com](https://htpasswd.utils.com/) pour créer l'utilisateur WUD comme dans l'exemple en dessous :  
<img width="512" height="400" alt="apr1-admin-wud" src="https://github.com/user-attachments/assets/4c8c8254-62c5-438f-be60-fcc1b6abd1d4" />


4. Créez y son `docker-compose.yml` et mettez y ce contenu en l'adaptant à votre besoin
```
services:
  whatsupdocker:
    image: getwud/wud
    container_name: wud
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock 
    ports:
      - 3030:3000
    environment:
      - WUD_WATCHER_LOCAL_CRON=<Votre-Cron>
      - WUD_AUTH_BASIC_ADMIN_USER=<Votre-admin-WUD>
      - WUD_AUTH_BASIC_ADMIN_HASH=<Hash-motdepasse-admin-wud>
      - WUD_TRIGGER_DOCKER_LOCAL_PRUNE=true
      - WUD_TRIGGER_DISCORD_MONDISCORD_URL=<votre-webhook-discord>
      - WUD_TRIGGER_DISCORD_MONDISCORD_MODE=simple
    labels:
      - "wud.watch.digest=true"
```

**Détails :**
  - `WUD_WATCHER_LOCAL_CRON` : L'heure à laquelle WUD va vérifier les mises à jours (ex : tout les jours à 01h du matin donne `WUD_WATCHER_LOCAL_CRON=0 1 * * *` )
  - `WUD_AUTH_BASIC_ADMIN_USER` : L'utilisateur admin pour l'interface web (ex : adminwud)
  - `WUD_AUTH_BASIC_ADMIN_HASH` : Le hash du mot de passe de l'utilisateur admin **Pensez à doubler les `$`** (ex : `$$apr1$$nW8LNJGMZ$$9KMQktDJJmw4n1MQyo7s7/`)
  - `WUD_TRIGGER_DOCKER_LOCAL_PRUNE` : Active le nettoyage des anciennes images et permet aussi d'activer l'application des mises à jours automatiquement !
  - `WUD_TRIGGER_DISCORD_MONDISCORD_URL` : Votre Webhook créé dans votre serveur discord dans `Paramètres du serveur` --> `Intégrations` --> `Webhooks`
  - `WUD_TRIGGER_DISCORD_MONDISCORD_MODE` : Format de notification discord
  - `wud.watch.digest=true` : Permet à WUD de regarder ses propres mises à jours

5. Déployez What's Up Docker
```
docker compose up -d
```

6. Pour vérifier que WUD fonctionne
```
docker ps -a
```

Pour être absolument sûr, regardez les logs
```
docker logs wud
```
