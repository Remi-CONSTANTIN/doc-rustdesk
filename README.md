# doc-rustdesk

## Objectif
Ce dépôt vise à fournir une base claire pour :
- Auto-héberger un relais RustDesk via la méthode officielle recommandée (Docker) afin de ne pas dépendre d’un service tiers
- Prendre la main à distance sur des machines via le relais RustDesk
- Appliquer des mesures de hardening adaptées à une exposition Internet

## Parcours recommandé
1. Suivre le tutoriel de déploiement  
[`Tutoriel-Deploiement-Rustdesk`](Tutoriel-Deploiement-Rustdesk.md) 
3. Suivre le tutoriel spécifique à CrowdSec pour renforcer la protection  
[`crowdsec-rustdesk`](crowdsec-rustdesk.md)
5. Mettez en place la mise à jours automatique  
[`rustdesk_auto-update`](rustdesk_auto-update.md)

## Disclaimer
- Même avec une bonne configuration / sécurisation d'un service exposé sur le web, vous ne serez pas immunisé contre toute intrusion. Toutes les mesures prises visent à réduire au maximum les risques
- Vous trouverez ici certains passages rédigés à l'aide d'une IA générative telle que Gemini. Cela n'empêche en aucun cas que toutes les manipulations ont été testées et approuvées par moi-même
