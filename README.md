# doc-rustdesk

Documentation en français pour déployer et sécuriser une instance **RustDesk auto-hébergée**.

## Contenu du dépôt

- [`Tutoriel-Deploiement-Rustdesk.md`](./Tutoriel-Deploiement-Rustdesk.md)  
  Guide complet pour installer et configurer un serveur RustDesk (ID + Relay) avec Docker, ouvrir les ports nécessaires, puis connecter un poste contrôlé et un poste contrôleur.

- [`crowdsec-rustdesk.md`](./crowdsec-rustdesk.md)  
  Guide de sécurisation avec CrowdSec pour détecter et bloquer les attaques visant une infrastructure RustDesk exposée sur Internet.

## Objectif

Ce dépôt vise à fournir une base claire pour :

- reprendre la main à distance sur des machines via RustDesk ;
- ne pas dépendre d’un service tiers ;
- appliquer des mesures de hardening adaptées à une exposition Internet.

## Parcours recommandé

1. Suivre le tutoriel de déploiement RustDesk.
2. Valider la connectivité (ports, ID, clé publique).
3. Appliquer les recommandations de sécurisation du tutoriel principal.
4. Mettre en place CrowdSec pour renforcer la protection.

## Avertissements importants

- N’exposez que les ports strictement nécessaires.
- Privilégiez une **whitelist d’IP publiques** si possible.
- Conservez la clé RustDesk (`id_ed25519.pub`) dans un emplacement sécurisé.
- Testez vos règles de sécurité avant mise en production.
