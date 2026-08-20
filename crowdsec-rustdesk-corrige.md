# Guide de Sécurisation : Protéger un déploiement Rustdesk (Docker) avec CrowdSec

Ce guide détaille la mise en place de CrowdSec sur un hôte Debian pour surveiller, détecter et bloquer en temps réel les attaques dirigées contre votre infrastructure Rustdesk auto-hébergée sous[...] 

# Disclaimer
Tutoriel rédigé à l'aide de Gemini, puis revu, testé et corrigé par Rémi CONSTANTIN

---

## Installation CrowdSec

1. Installer le dépôt CrowdSec sur la machine à protéger
```
curl -s https://install.crowdsec.net | sudo sh
```

2. Installer CrowdSec
```
apt install crowdsec
```

3. Si vous aviez déjà des services sur la machine lors de l'installation de CrowdSec, il est possible que des collections aient été téléchargées automatiquement.
Pour vérifier cela : 
```
cscli collections list
```

4. Pour finir l'installation, je vous recommande vivement d'installer tout de suite le "Bouncer" iptables afin que CrowdSec puisse bannir automatiquement s'il détecte une activité suspecte
```
apt install crowdsec-firewall-bouncer-iptables -y
```

## Configuration

Par défaut, le vigile ne regarde pas le trafic destiné à vos conteneurs. Il faut l'obliger à surveiller le réseau spécifique de Rustdesk.

1. Ouvrez le fichier de configuration
```
nano /etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml
```

2. Modifiez cette section et décommentez `FORWARD` et `DOCKER-USER`
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
et
```
systemctl restart docker
```

## Piste d'amélioration
Ajouter cette machine au dashboard CrowdSec Web officiel

