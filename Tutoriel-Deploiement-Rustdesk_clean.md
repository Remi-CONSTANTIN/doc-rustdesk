# Tutoriel Complet : Auto-héberger sa propre instance Rustdesk

Ce guide permet de déployer un serveur Rustdesk sécurisé servant de relais afin de prendre la main sur un ordinateur distant.  

L’intérêt de Rustdesk réside dans plusieurs points :
- Peut être auto-hébergé afin de ne pas dépendre d'un service tiers, parfois payant, tel que TeamViewer ou AnyDesk
- Compatible Linux, là où Google Remote Desktop ne l'est pas
- Permet de n'utiliser qu'un client portable ou une interface web pour prendre la main (pratique si vous n'avez pas les droits d'administrateur sur la machine pour prendre la main)

Aucune interface web n'est ici configurée car nous partons du principe que vous n'avez que quelques machines à superviser. En effet, elle ne ferait qu'ajouter de la surface d'attaque à votre infrastructure.

# Disclaimer
Tutoriel rédigé et testé par Rémi CONSTANTIN et aidé par Gemini

# Sources
[Documentation officielle sur l'auto-hébergement Rustdesk](https://rustdesk.com/docs/en/self-host/)

---

# Prérequis
- Une machine (de préférence Linux) avec un accès administrateur où il est possible d'installer Docker (pour l'installer sur Linux c'est [ICI](https://docs.docker.com/engine/install/))
- Avoir accès aux redirections de port sur son routeur
- (Optionnel mais très recommandé) Placez votre machine dans une DMZ car nous allons l'exposer à Internet

# Mise en place
Nous verrons ici l'installation des services Rustdesk sur un LXC Debian 13 sur l’hyperviseur Proxmox. À vous d'adapter les manipulations à votre contexte technique.

## Installation Rustdesk

1. Pour commencer, assurez-vous d'avoir installé Docker sur votre machine en testant une commande (`docker ps` par exemple)

2. Créez un répertoire de travail où vous le souhaitez
```
mkdir /opt/rustdesk
```
3. Créez-y le docker compose de rustdesk avec la configuration suivante (à adapter à votre contexte, bien sûr)
```
nano docker-compose.yml
```
**Configuration**
```
# Création d'un réseau interne isolé pour que les deux conteneurs communiquent entre eux
networks:
  rustdesk-net:
    external: false

services:
  # hbbs = RustDesk ID Server (Gère les connexions réseau, l'annuaire et les identifiants)
  hbbs:
    container_name: hbbs
    image: rustdesk/rustdesk-server:latest
    # RAPPEL : Remplace TON_IP_PUBLIQUE par l'IP fixe de ta box internet
    # L'argument "-k _" force la création et l'utilisation de la clé de chiffrement (id_ed25519.pub)
    command: hbbs -r TON_IP_PUBLIQUE:21117 -k _
    volumes:
      # Rapatrie la base de données et les clés de chiffrement dans le dossier ./data de ton LXC
      - ./data:/root
    networks:
      - rustdesk-net
    # Mapping des ports de l'ID Server vers l'extérieur
    ports:
      - 21115:21115      # TCP : Service de base (Test NAT)
      - 21116:21116      # TCP : Connexion réseau
      - 21116:21116/udp  # UDP : Indispensable pour la vitesse et le "Heartbeat" (battement de coeur)
    depends_on:
      - hbbr             # S'assure que le relais démarre avant le serveur ID
    restart: unless-stopped
    
    # --- BLOC SÉCURITÉ DOCKER ---
    security_opt:
      - no-new-privileges:true # Empêche un pirate d'obtenir de nouveaux droits dans le conteneur
    cap_drop:
      - ALL                    # Retire tous les droits d'administration (root) au conteneur
    deploy:
      resources:
        limits:
          cpus: '0.5'          # Bridage : utilise maximum 50% d'un thread CPU de ton Proxmox
          memory: 256M         # Bridage : limite la consommation RAM à 256 Mo pour éviter les crashs

  # hbbr = RustDesk Relay Server (Gère le flux vidéo/clavier quand la connexion directe P2P échoue)
  hbbr:
    container_name: hbbr
    image: rustdesk/rustdesk-server:latest
    # Le relais est lui aussi sécurisé avec la clé de chiffrement obligatoire
    command: hbbr -k _
    volumes:
      - ./data:/root
    networks:
      - rustdesk-net
    # Mapping du port Relais vers l'extérieur
    ports:
      - 21117:21117      # TCP : Service de relais de l'écran
    restart: unless-stopped
    
    # --- BLOC SÉCURITÉ DOCKER ---
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
```

4. Appliquez le docker compose
```
docker compose up -d
```

5. Vérifiez que les conteneurs sont montés
```
docker ps
```

Nous avons bien les deux conteneurs `hbbs` et `hbbr` UP

6. Récupérez votre clé qui vous permettra de vous connecter à vos machines et stockez-la dans un endroit sécurisé (coffre à mots de passe comme KeePass par exemple)
```
cat /opt/rustdesk/data/id_ed25519.pub
```
C'est tout pour la partie système

---

## Configuration du routeur
Une fois le service déployé, il nous faut l'exposer au web dans l'optique d'y accéder depuis un autre lieu.  
Les prochaines étapes dépendront de votre matériel réseau.

Voici les redirections de ports à configurer : 
|Port|Protocole|Redirigé vers|Port machine Rustdesk|À quoi ça sert|
|----|---------|--------------------------|---------------------|--------------|
|21115|TCP|IP machine Rustdesk|21115|Authentification (hbbs)|
|21116|TCP et UDP|IP machine Rustdesk|21116|Connexion et Heartbeat (hbbs)|
|21117|TCP|IP machine Rustdesk|21117|Service de Relais (hbbr)|

> [!warning]
> Si vous le pouvez, configurez une whitelist d'IP publiques autorisées à accéder à ces redirections de port afin de limiter grandement la surface d'attaque

La configuration du relais est terminée

---

## Configuration de la prise en main
Maintenant que vous avez votre propre relais Rustdesk, passons à la configuration du client sur le contrôlé et le contrôleur.

### Configuration d'un ordinateur contrôlé
1. Téléchargez le client Rustdesk sur le [GitHub Officiel](https://github.com/rustdesk/rustdesk/releases/latest) en fonction de votre OS/distribution/architecture

2. Exécutez le fichier téléchargé et installez-le si vous le souhaitez

Voici les avantages de l'installation :
- Le démarrage automatique (Accès non surveillé) : le logiciel se lance tout seul dès l'allumage de l'ordinateur. Vous n'avez pas besoin que quelqu'un soit physiquement présent devant le PC pour autoriser la connexion.
- Le contrôle des fenêtres d'administration : si vous utilisez la version portable, certaines opérations (installer un logiciel, modifier un paramètre système) peuvent nécessiter des droits administrateur qui ne seront pas disponibles sans installation.
- L'accès à l'écran de verrouillage : si vous devez redémarrer un PC à distance, la version installée vous permettra de voir l'écran de connexion et de saisir le mot de passe de session pour reprendre la main.
- Le maintien de la configuration : votre IP publique personnalisée, votre clé de chiffrement et le mot de passe fixe que vous allez définir seront sauvegardés "en dur" dans le système et ne s'effaceront pas à chaque redémarrage.

(source : Gemini)

3. Dans le client, cliquez sur les 3 petits points à côté de votre `ID` et allez dans `Réseau`

4. Cliquez sur le bouton `Déverrouiller les paramètres réseau` et complétez les champs :
- `ID` (ID Server) : saisissez votre IP publique fixe
- `Key` : collez votre clé secrète (trouvée dans `/opt/rustdesk/data/id_ed25519.pub`)

> [!warning]
> Laissez les champs "Serveur Relais" et "Serveur API" complètement vides ! RustDesk est intelligent, il trouvera le port 21117 tout seul

5. Cliquez sur `Appliquer`, laissez le client ouvert et passez sur l'ordinateur contrôleur

### Configuration d'un ordinateur contrôleur
La configuration est presque la même que pour le contrôlé à quelques détails près.

1. Comme pour l'ordinateur contrôlé, téléchargez le client sur le [GitHub Officiel](https://github.com/rustdesk/rustdesk/releases/latest) en fonction de votre OS/distribution/architecture

2. Exécutez le fichier téléchargé et installez-le si vous le souhaitez (il y a moins d’intérêt à le faire sur le PC contrôleur, à part peut-être pour les mises à jour plus fluides)

3. Dans le client, cliquez sur les 3 petits points à côté de votre `ID` et allez dans `Réseau`

4. Cliquez sur le bouton `Déverrouiller les paramètres réseau` et complétez les champs :
- `ID` (ID Server) : saisissez votre IP publique fixe
- `Key` : collez votre clé secrète (trouvée dans `/opt/rustdesk/data/id_ed25519.pub`)

### Test de connexion
1. Comme avec TeamViewer ou AnyDesk, cherchez l'`ID` sur le client du PC contrôlé et saisissez-le sur le PC contrôleur

2. Entrez le mot de passe affiché sur le client contrôlé (vous pouvez le fixer dans les paramètres du client)

Vous devriez maintenant avoir la main sur votre ordinateur ! 

> [!warning]
> Vous rencontrerez sans doute des problèmes d'affichage avec les distributions Linux qui utilisent le serveur d'affichage `Wayland` (telles que Fedora KDE)

---

# Sécurisation
Maintenant que vous avez un relais fonctionnel, il est très fortement recommandé de faire du hardening.

Voici quelques idées :

- **DMZ**  
Comme dit au début du tutoriel, l'idéal est de placer le serveur dans votre DMZ afin de préserver vos machines locales d'une éventuelle infection par déplacement latéral.

- **Whitelist**  
Comme dit plus tôt dans le tutoriel, essayez de filtrer l'accès au serveur en limitant les IP publiques autorisées à passer par les redirections de ports en y mettant une whitelist (possible sur la plupart des routeurs ou via un pare-feu matériel).

- **Firewall**  
Afin d'éviter les déplacements latéraux provenant d'autres machines de la DMZ, fermez le maximum de ports avec un pare-feu. Il est d'ailleurs possible de faire la whitelist d'IP publiques par ce biais.

> [!caution]
> N'utilisez pas de pare-feux directement sur le système de la machine (tel qu'UFW, iptables, nftables etc.) car Docker installe des règles de routage qui les contournent.
> Privilégiez plutôt des solutions qui se placent au-dessus du serveur comme Proxmox Firewall si c'est une machine virtuelle

- **Anti-brute-force**  
De nombreuses solutions permettent de détecter les attaques sur votre machine et de bannir en conséquence, comme Fail2Ban ou CrowdSec (qui est une version moderne/améliorée).  
Ces solutions sont d'autant plus utiles si vous n'avez pas mis en place de whitelist !
