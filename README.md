# doc-rustdesk

## Contenu du dépôt

- [`Tutoriel-Deploiement-Rustdesk.md`](./Tutoriel-Deploiement-Rustdesk.md)  
  Guide complet pour installer et configurer un serveur RustDesk (ID + Relay) avec Docker, ouvrir les ports nécessaires, puis connecter un poste contrôlé et un poste contrôleu

- [`crowdsec-rustdesk.md`](./crowdsec-rustdesk.md)  
  Guide de sécurisation avec CrowdSec pour détecter et bloquer les attaques visant une infrastructure RustDesk exposée sur Internet.

## Objectif
Ce dépôt vise à fournir une base claire pour :
- Auto-heberger un relais Rustdesk via la méthode officielle recommendée (docker) afin de ne pas dépendre d’un service tiers
- Prendre la main à distance sur des machines via le relai RustDesk
- Appliquer des mesures de hardening adaptées à une exposition Internet

## Parcours recommandé
1. Suivre le tutoriel de déploiement [`crowdsec-rustdesk.md`](./crowdsec-rustdesk.md) 
2. Valider la connectivité (ports, ID, clé publique)
3. Appliquer les recommandations de sécurisation du tutoriel principal
4. Suivre le tutoriel spécifique à CrowdSec pour renforcer la protection [`crowdsec-rustdesk.md`](./crowdsec-rustdesk.md)

## Disclaimer
- Même avec une bonne configuration / sécurisation d'un service exposé a web, vous ne serez pas immunisé à toute intrusion. Toutes les mesures prisent visent à réduire au maximum les risques
- Vous trouverez ici certains passages rédigés à l'aide d'IA générative tel que Gemini. Cela n'empêche en aucun cas que toutes les manipulations ont été testées et approuvées par moi-même. Vous ne trouverez aucune documentation bêtement copiée sans avoir été relue
