# Tutoriel Complet : Auto-herberger sa propre instance Rustdesk

Ce guide permet de déployer un serveur Rustdesk sécurisé servant de relais afin de prendre la main sur un ordinateur distant.  

L’intérêt de Rustdesk réside dans plusieurs points : 
- Peut être auto-herbergé afin de ne pas dépendre d'un service tiers parfois payant tel que Teamviewer ou Anydesk
- Compatible Linux, là où Google Remote Desktop ne l'est pas
- Permet de n'utiliser qu'un client portable ou une interface web pour prendre la main (pratique si vous n'avez pas les droits administrateurs sur la machine pour prednre la main)

 Aucune interface web n'est ici configurée car nous partons du principe que vous n'avez que quelques machines à superviser. En effet, elle ne ferait qu'ajouter que de la surface d'attaque à votre instance, en plus de nécessiter un proxy.

# Disclaimer
Tutoriel rédigé + testé par Rémi CONSTANTIN et aidé par Gemini

# Sources
[Documentation officielle sur l'auto-hebergement Rustdesk](https://rustdesk.com/docs/en/self-host/)

---

# Prérequis
- Une machine(de préférence Linux) avec un accès admin où il est possible d'installer Docker (pour l'installer sur Linux c'est [ICI](https://docs.docker.com/engine/install/))
- Avoir accès aux redirections de port sur son routeur
- (Optionnel mais très recommandé) Placez votre machine dans une DMZ car nous allons l'exposer à internet

# Mise en place
Nous verrons ici l'installation des services Rustdesk sur un LXC Debian 13 sur l’hyperviseur Proxmox. A vous d'adapter les manipulations à votre contexte technique.

## Installation Rustdesk

1. Pour commencer, assurez vous d'avoir installer Docker sur vote machine en testant une commande (`docker ps` par exemple)

2. Créez vous un répertoire de travail où vous voulez
```
mkdir /opt/rustdesk
```
3. Créez-y le docker compose de rustdesk avec la configuration suivante (à adapter à votre contexte, bien sûr)
```
nano docker-compose.yml
```
**Configuration**
```
# Creation d'un reseau interne isole pour que les deux conteneurs communiquent entre eux
networks:
  rustdesk-net:
    external: false

services:
  # hbbs = RustDesk ID Server (Gere les connexions reseau, l'annuaire et les identifiants)
  hbbs:
    container_name: hbbs
    image: rustdesk/rustdesk-server:latest
    # RAPPEL : Remplace TON_IP_PUBLIQUE par l'IP fixe de ta box internet
    # L'argument "-k _" force la creation et l'utilisation de la cle de chiffrement (id_ed25519.pub)
    command: hbbs -r TON_IP_PUBLIQUE:21117 -k _
    volumes:
      # Rapatrie la base de donnees et les cles de chiffrement dans le dossier ./data de ton LXC
      - ./data:/root
    networks:
      - rustdesk-net
    # Mapping des ports de l'ID Server vers l'exterieur
    ports:
      - 21115:21115      # TCP : Service de base (Test NAT)
      - 21116:21116      # TCP : Connexion reseau
      - 21116:21116/udp  # UDP : Indispensable pour la vitesse et le "Heartbeat" (battement de coeur)
    depends_on:
      - hbbr             # S'assure que le relais demarre avant le serveur ID
    restart: unless-stopped
    
    # --- BLOC SECURITE DOCKER ---
    security_opt:
      - no-new-privileges:true # Empeche un pirate d'obtenir de nouveaux droits dans le conteneur
    cap_drop:
      - ALL                    # Retire tous les droits d'administration (root) au conteneur
    deploy:
      resources:
        limits:
          cpus: '0.5'          # Bridage : utilise maximum 50% d'un thread CPU de ton Proxmox
          memory: 256M         # Bridage : limite la consommation RAM a 256 Mo pour eviter les crashs

  # hbbr = RustDesk Relay Server (Gere le flux video/clavier quand la connexion directe P2P echoue)
  hbbr:
    container_name: hbbr
    image: rustdesk/rustdesk-server:latest
    # Le relais est lui aussi securise avec la cle de chiffrement obligatoire
    command: hbbr -k _
    volumes:
      - ./data:/root
    networks:
      - rustdesk-net
    # Mapping du port Relais vers l'exterieur
    ports:
      - 21117:21117      # TCP : Service de relais de l'ecran
    restart: unless-stopped
    
    # --- BLOC SECURITE DOCKER ---
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

3. Appliquez le docker compose
```
docker compose up -d
```

4. Vérifiez que les conteneurs sont montés
```
docker ps
```
<img width="2518" height="253" alt="docker-ps_rustdesk" src="https://github.com/user-attachments/assets/253f508e-4585-4307-8855-75e8c19df17d" />  
Nous avons bien les deux conteneurs `hbbs` et `hbbr` UP

5. Récupérez votre clé qui vous permettra de vous connecter à vos machines et stockez la dans un endroit sécurisé (coffre à mot de passe comme Keepass par exemple)
```
cat /opt/rustdesk/data/id_ed25519.pub
```
C'est tout pour la partie système

---

## Configuration du routeur
Une fois le service déployé, il nous faut l'exposer au web dans l'optique d'y accéder depuis un autre lieu.  
Les prochaines étapes dépendront de votre matériel réseau.

Voici les redirections de ports à configurer : 
|Port|Protocole|Redirigé vers|Port machine Rustdesk|A quoi ça sert|
|----|---------|--------------------------|---------------------|--------------|
|21115|TCP|IP machine Rustdesk|21115|Authentification (hbbs)|
|21116|TCP et UDP|IP machine Rustdesk|21116|Connexion et Heartbeat (hbbs)|
|21117|TCP|IP machine Rustdesk|21117|Service de Relais (hbbr)|

> [!warning]
> Si vous le pouvez, configurez une whitelist d'IP publique autorisées à accéder à ces redirections de port afin de limiter grandement la surface d'attaque

La configuration du relais est terminée

---

## Configuration de la prise en main
Maintenant que vous avez votre propre relais Rustdesk, passons à la configuration du client sur le controlé et le contrôleur 

### Configuration d'un ordinateur contrôlé
1. Télécharger le client Rustdesk sur le [Github Officiel](https://github.com/rustdesk/rustdesk/releases/latest) en fonction de votre OS/distribution/architecture

2. Exécutez le fichier téléchargé et installez le si vous voulez

Voici les avantages de l'installation :
- Le démarrage automatique (Accès non surveillé) : Le logiciel se lance tout seul dès l'allumage de l'ordinateur. Tu n'as pas besoin que quelqu'un soit physiquement présent devant le PC Fedora pour double-cliquer sur l'application et te donner l'ID.
- Le contrôle des fenêtres d'administration : C'est le point le plus important. Si tu utilises la version portable et que tu tentes d'installer un logiciel ou de modifier un paramètre réseau à distance, une fenêtre te demandant le mot de passe administrateur va s'ouvrir. Sans installation de RustDesk, ton écran va se bloquer et tu ne pourras pas cliquer (c'est une sécurité de Linux/Windows). L'installation donne à RustDesk les droits d'interagir avec ces fenêtres sécurisées.
- L'accès à l'écran de verrouillage : Si tu dois redémarrer ton PC Fedora à distance, la version installée te permettra de voir l'écran de connexion et de taper ton mot de passe de session pour rouvrir le bureau. Une version portable se couperait au redémarrage et tu perdrais définitivement la main.
- Le maintien de la configuration : Ton IP publique personnalisée, ta clé de chiffrement et le mot de passe fixe que tu vas définir seront sauvegardés "en dur" dans le système et ne s'effaceront pas par erreur.
(source : gemini)

3. Dans le client, cliquez sur les 3 petits points à côté de votre `ID` et allez dans `Réseau`

4. Cliquez sur le bouton `Déverrouiller les paramètres réseau` et complétez les champs :
- `ID` (ID Server) : tape ton IP publique fixe
- `Key` : colle ta clé secrète (trouvée dans `/opt/rustdesk/data/id_ed25519.pub`)

> [!warning]
> Laisse les champs "Serveur Relais" et "Serveur API" complètement vides ! RustDesk est intelligent, il trouvera le port 21117 tout seul

5. Cliquez sur `Appliquer`, laissez le client ouvert et passez sur l'ordinateur contrôleur

### Configuration d'un ordinateur contrôleur
La configuration est presque la même que pour le contrôlé à quelques détails près.

1. Comme pour l'ordinateur contrôlé, téléchargez le client sur le [Github Officiel](https://github.com/rustdesk/rustdesk/releases/latest) en fonction de votre OS/distribution/architecture

2. Exécutez le fichier téléchargé et installez le si vous voulez (il y a moins d’intérêt à le faire sur le PC contrôleur, à part peut être pour les mises à jours plus fluides)

3. Dans le client, cliquez sur les 3 petits points à côté de votre `ID` et allez dans `Réseau`

4. Cliquez sur le bouton `Déverrouiller les paramètres réseau` et complétez les champs :
- `ID` (ID Server) : tape ton IP publique fixe
- `Key` : colle ta clé secrète (trouvée dans `/opt/rustdesk/data/id_ed25519.pub`)

### Test de connexion
1. Comme avec Teamviwer ou Anydesk, allez cherchez l'`ID` sur le client du pc contrôlé et entrez le sur le pc contrôleur

2. Entrez le mot de passe affiché sur le client contrôlé  (vous pouvez le fixer dans les paramères du client)

Vous devriez maintenant avoir la main sur votre ordinateur ! 

> [!warning]
> Vous rencontrerez sans doute des problèmes d'affichage avec les distributions linux qui utilisent le serveur d'affichage `Wayland` (tel que Fedora KDE)

---

# Sécurisation
Maintenant que vous avez un relais fonctionnel, il est très fortement recommandé de faire du hardening.  

Voici quelques idées :

**Whitelist**  
Comme dis plus tôt dans le tutoriel, essayez de filtrer l'accès au serveur en limitant les IP publiques autorisées à passer par les redirections de ports en y placant une whitelist (possible sur l'envirennement unifi d'Ubiquiti)

**Firewall**  
Afin d'éviter les déplacement horizontaux  provenant d'autres machines de la DMZ fermez le maximum de port avec un pare-feu. Il est d'ailleurs possible de faire la whitelist d'IP publique par ce biais si pas possible sur le routeur.

> [!caution]
> N'utilisez pas de pare-feux directement sur le système de la machine (tel qu'UFW, IPtables, NFtables etc...) car Docker installe des règles de routages qui les bypassent.
> Privilégiez plutôt des solutions qui se placent au dessus du serveur comme Proxmox Firewall si c'est une machine virtuelle

**Anti-brutforces**  
De nombreuses solutions permettent de détecter les attaques sur votre machine et de bannir en conséquence comme Fail2Ban ou CrowdSec qui est sa version améliorée.  
Ces solutions sont d'autant plus utiles si vous n'avez pas fais de whitelist !
