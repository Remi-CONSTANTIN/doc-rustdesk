# Guide de Sécurisation : Protéger un déploiement Rustdesk (Docker) avec CrowdSec

Ce guide détaille la mise en place de CrowdSec sur un hôte Debian pour surveiller, détecter et bloquer en temps réel les attaques dirigées contre votre infrastructure Rustdesk auto-hébergée sous Docker

# Disclaimer
Tutoriel rédigé à l'aide de Gemini puis revu + testé + corrigé par Rémi CONSTANTIN

---

## Installation Crowdsec

1. Installer le dépôt CrowdSec sur la machine à protéger
```
curl -s https://install.crowdsec.net | sudo sh
```

2. Installer CrowdSec
```
apt install crowdsec
```

3. Si vous aviez déjà des services sur la machine lors de l'installation de CrowdSec, alors il est possible que des collections aient automatiquement été téléchargées.
Pour vérifier cela : 
```
cscli collections list
```

4. Ajoutons une collection contre les scans de port
```
cscli collections install crowdsecurity/iptables
```

5. Pour finir l'installation , je vous recommande vivement d'installer tout de suite le "Bouncer" iptables afin que CrowdSec puisse bannir automatiquement s'il détecte une activité suspecte
```
apt install crowdsec-firewall-bouncer-iptables -y
```

## Configuration

Par défaut, le vigile ne regarde pas le trafic destiné à vos conteneurs. Il faut l'obliger à surveiller le réseau spécifique de RustDesk

1. Ouvrez le fichier de configuration
```
nano /etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml
```

2. Modifiez cette section et décommenter `FORWARD` et `DOCKER-USER`
```
iptables_chains:
  - INPUT
  - FORWARD
  - DOCKER-USER
```

3. Redémarrez les services pour appliquer la configuration
```
systemctl restart crowdsec-firewall-bouncer
```
ET
```
systemctl restart docker
```

Et c'est tout ce que nous pouvons faire pour la protection de notre instance avec CrowdSec

## Piste d'amélioration
Ajouter cette machine au dashboard Crowdsec Web officiel
