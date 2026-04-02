# EC04 — UrbanHub Park

Guide de révision et d’exécution pour **EC04.1 (implémentation Cloud)** et **EC04.2 (sécurisation Cloud)**.

> **Objectif** : avoir un support clair, lisible sur GitHub, rapide à consulter pendant l’examen.
>
> **Région imposée** : `eu-west-3`.

---

## Sommaire

- [Vue d’ensemble](#vue-densemble)
- [EC04.1 — Refaire rapidement la partie 1](#ec041--refaire-rapidement-la-partie-1)
  - [Architecture attendue](#architecture-attendue)
  - [Ordre d’exécution recommandé](#ordre-dexécution-recommandé)
  - [Étapes détaillées](#étapes-détaillées)
  - [Captures à rendre](#captures-à-rendre)
  - [Pannes fréquentes](#pannes-fréquentes)
- [EC04.2 — Sécuriser l’environnement Cloud](#ec042--sécuriser-lenvironnement-cloud)
  - [Méthode de travail](#méthode-de-travail)
  - [Diagnostic par couches](#diagnostic-par-couches)
  - [Actions de sécurisation probables](#actions-de-sécurisation-probables)
  - [Monitoring et alerting](#monitoring-et-alerting)
  - [Justifications techniques prêtes à l’emploi](#justifications-techniques-prêtes-à-lemploi)
  - [Preuves à capturer](#preuves-à-capturer)
- [Checklist finale](#checklist-finale)
- [Commandes utiles](#commandes-utiles)

---

## Vue d’ensemble

### Partie 1 — Implémentation Cloud
Construire une architecture AWS **fonctionnelle** pour UrbanHub :

- application accessible via **ALB**,
- instances **EC2** dans un **Auto Scaling Group**,
- base **MariaDB sur Amazon RDS**,
- architecture en **eu-west-3**,
- routage et sécurité cohérents.

### Partie 2 — Sécurisation Cloud
Analyser, corriger et prévenir les vulnérabilités de l’environnement créé en P1 :

- diagnostic des risques,
- IAM / rôles / permissions / politiques,
- bonnes pratiques provider,
- monitoring / alerting,
- durcissement,
- preuves et justification technique.

---

# EC04.1 — Refaire rapidement la partie 1

## Architecture attendue

### Exigences clés

- Région unique : `eu-west-3`
- **ALB** devant les EC2
- **Target Group** HTTP:80
- **ASG** : min `1`, max `8`, desired `1`
- **Target tracking CPU** : `50 %`
- **RDS MariaDB** privé
- Base **non accessible directement depuis Internet**
- VPC : `10.0.0.0/16`
- Subnets :
  - public AZ1 : `10.0.1.0/24`
  - public AZ2 : `10.0.2.0/24`
  - privé AZ1 : `10.0.3.0/24`
  - privé AZ2 : `10.0.4.0/24`
- Subnets publics :
  - auto-assign public IPv4 **activé**
  - DNS public **activé**
- Rôle EC2 avec **AmazonRDSReadOnlyAccess**

---

## Ordre d’exécution recommandé

1. **VPC + subnets + DNS**
2. **Internet Gateway + route table publique**
3. **Security Groups**
4. **RDS MariaDB + DB subnet group privé**
5. **IAM Role EC2**
6. **Launch Template**
7. **Target Group**
8. **ALB**
9. **ASG**
10. **Tests + captures**

> **Conseil examen** : ne pas improviser l’ordre. Suivre la séquence ci-dessus.

---

## Étapes détaillées

### 1) VPC et sous-réseaux

Créer le VPC `10.0.0.0/16`.

Activer :

- `DNS resolution`
- `DNS hostnames`

Créer les 4 subnets :

| Type | CIDR | AZ |
|---|---:|---|
| Public 1 | `10.0.1.0/24` | eu-west-3a |
| Public 2 | `10.0.2.0/24` | eu-west-3b |
| Privé 1 | `10.0.3.0/24` | eu-west-3a |
| Privé 2 | `10.0.4.0/24` | eu-west-3b |

Dans les **2 subnets publics** :

- activer **Attribution automatique d’IPv4 publique**.

---

### 2) Internet Gateway et routage

Créer une **Internet Gateway** et l’attacher au VPC.

Créer une **route table publique** :

- `10.0.0.0/16 -> local`
- `0.0.0.0/0 -> Internet Gateway`

Associer cette table aux **2 subnets publics**.

Les subnets privés restent sans route Internet.

---

### 3) Security Groups

#### SG ALB

- **Inbound** : HTTP `80` depuis `0.0.0.0/0`
- **Outbound** : tout

#### SG EC2

- **Inbound** : HTTP `80` depuis le **SG ALB**
- **Inbound** : SSH `22` depuis **votre IP**
- **Outbound** : tout

#### SG RDS

- **Inbound** : MariaDB / MySQL `3306` depuis le **SG EC2 uniquement**
- **Outbound** : par défaut

> Ne jamais mettre `3306` ouvert à `0.0.0.0/0`.

---

### 4) RDS MariaDB

Créer une base **MariaDB** avec :

- **DB identifier** : `urbanhub-db`
- **username** : `admin`
- **password** : `SuperPassword123!` *(ou autre, mais garder la cohérence avec le user data)*
- **Public access** : `No`
- **Security Group** : `urbanhub-sg-rds`

Créer un **DB subnet group** avec **uniquement** :

- `10.0.3.0/24`
- `10.0.4.0/24`

Attendre l’état **Available** avant de continuer.

---

### 5) IAM Role EC2

Créer un rôle IAM pour **EC2**.

Attacher la policy :

- `AmazonRDSReadOnlyAccess`

Nom conseillé :

- `urbanhub-ec2-role`

---

### 6) Launch Template

Choix principaux :

- AMI Ubuntu Server
- type `t3.micro`
- key pair
- SG : `urbanhub-sg-ec2`
- IAM instance profile : `urbanhub-ec2-role`

Ne pas figer :

- le subnet
- la zone de disponibilité

Le choix des subnets sera fait par l’ASG.

#### User data

```bash
#!/bin/bash
export MASTER_DB_USER=admin
export MASTER_DB_PASSWORD='SuperPassword123!'
export AWS_REGION='eu-west-3'
curl -fsSL https://raw.githubusercontent.com/dahut87/urban_1/refs/heads/main/install.sh | bash
```

---

### 7) Target Group

Créer un Target Group :

- Type : `Instances`
- Protocole : `HTTP`
- Port : `80`
- VPC : votre VPC

#### Health check

- Protocol : `HTTP`
- Path : `/health.php`
- Success code : `200`

---

### 8) ALB

Créer un **Application Load Balancer** :

- Scheme : `internet-facing`
- Listener : `HTTP:80`
- Subnets : **2 subnets publics**
- SG : `urbanhub-sg-alb`
- Action par défaut : forward vers `urbanhub-tg`

---

### 9) Auto Scaling Group

Créer un ASG :

- Launch Template : celui créé plus haut
- VPC : votre VPC
- Subnets : **2 subnets publics**
- Attach to existing load balancer / target group : `urbanhub-tg`

#### Capacité

- desired : `1`
- min : `1`
- max : `8`

#### Scaling policy

- type : **Target tracking**
- metric : **Average CPU Utilization**
- target value : `50`

#### Health checks

- activer les checks **EC2** + **ELB**
- grace period : `300` secondes

---

## Captures à rendre

### Obligatoires

- capture de l’application UrbanHub fonctionnelle,
- capture de la commande CloudShell qui affiche le DNS public de l’ALB + résultat,
- capture de `systemctl status apache2`.

### Bonus très utiles

- ALB + DNS,
- Target Group avec cible **healthy**,
- RDS avec **Public access = No**,
- ASG avec min / desired / max,
- scaling CPU `50 %`,
- subnets publics / privés,
- SG EC2 / SG RDS.

---

## Pannes fréquentes

| Symptôme | Cause probable | Correction rapide |
|---|---|---|
| `502 Bad Gateway` sur ALB | target group vide ou instance non healthy | vérifier ASG, target group, health check |
| target group = `0 cible` | ASG non attaché au TG | rattacher `urbanhub-tg` à l’ASG |
| pas de SSH | pas de key pair ou pas d’IP publique | corriger le Launch Template / vérifier auto-assign public IPv4 |
| pas d’IP publique | subnet public mal configuré | activer auto-assign public IPv4 sur le subnet public |
| RDS inaccessible | SG RDS trop strict ou mauvais subnet group | vérifier port `3306` depuis SG EC2 et DB subnet group privé |
| script user data échoue | rôle IAM absent ou mauvais mot de passe DB | vérifier rôle EC2 + user data + cohérence des credentials |
| EC2 répond en direct impossible | normal si SG EC2 accepte seulement HTTP depuis SG ALB | tester via le **DNS de l’ALB**, pas via l’EC2 directe |

---

# EC04.2 — Sécuriser l’environnement Cloud

## Méthode de travail

### Logique à suivre

Faire la P2 **par passes** :

1. **Diagnostic**
2. **Corrections IAM**
3. **Corrections réseau / exposition**
4. **Durcissement des ressources**
5. **Monitoring / alerting**
6. **Captures et justification**

> Ne pas partir dans tous les sens. Toujours raisonner par couches.

---

## Diagnostic par couches

## 1) IAM

Questions à se poser :

- les rôles IAM sont-ils **minimaux** ou trop larges ?
- qui peut faire quoi ?
- y a-t-il des permissions inutiles ?
- la policy EC2 est-elle justifiée ?

### Attendu défendable

- garder **seulement** ce qui est nécessaire au fonctionnement ;
- éviter les permissions admin globales ;
- documenter pourquoi telle policy reste nécessaire.

---

## 2) Réseau

Questions à se poser :

- quelles ressources sont exposées à Internet ?
- est-ce voulu ?
- le trafic suit-il le bon chemin ?

### Chemin correct

**Internet -> ALB -> EC2 -> RDS**

### Ce qui doit être vrai

- ALB exposé publiquement : **oui**
- EC2 exposées directement en HTTP depuis Internet : **non**
- RDS exposée publiquement : **non**
- SSH limité à votre IP : **oui**

---

## 3) Exposition

Vérifier :

- subnets publics = uniquement pour ALB + EC2 si nécessaire,
- base dans les subnets privés,
- SG EC2 ne laisse pas entrer `80` depuis `0.0.0.0/0`,
- SG RDS n’ouvre jamais `3306` au monde.

---

## 4) Monitoring

Questions à traiter :

- comment détecter une panne ?
- comment détecter une saturation ?
- comment prouver que l’infra est suivie ?

---

## 5) Durcissement

Questions à se poser :

- qu’est-ce qu’on peut fermer ?
- qu’est-ce qu’on peut restreindre ?
- qu’est-ce qu’on peut surveiller ?
- qu’est-ce qu’on peut justifier ?

---

## Actions de sécurisation probables

## IAM

### À faire

- vérifier le rôle EC2 ;
- garder uniquement la policy nécessaire ;
- éviter toute policy trop large si elle existe ;
- vérifier qu’aucun utilisateur humain n’a des droits excessifs inutiles.

### Justification prête

> Le rôle IAM de l’instance est limité à la lecture RDS nécessaire au script d’installation. Cela permet de respecter le principe du moindre privilège tout en conservant le fonctionnement applicatif.

---

## Security Groups

### À garder / corriger

#### SG ALB

- HTTP 80 depuis Internet : **oui**

#### SG EC2

- HTTP 80 depuis **SG ALB uniquement** : **oui**
- SSH 22 depuis **votre IP uniquement** : **oui**
- supprimer toute ouverture large inutile

#### SG RDS

- 3306 depuis **SG EC2 uniquement** : **oui**
- aucun accès Internet direct : **oui**

### Justification prête

> L’exposition réseau a été réduite au strict nécessaire. L’ALB est le seul point d’entrée public. Les instances EC2 ne reçoivent le trafic applicatif que depuis l’ALB, et la base RDS n’est accessible que depuis les EC2 de l’application.

---

## Subnets et exposition

### À vérifier

- ALB dans subnets publics,
- RDS dans subnets privés,
- route Internet seulement là où c’est justifié,
- EC2 avec IP publique uniquement si nécessaire à l’administration / à l’exercice.

### Justification prête

> La séparation public / privé réduit la surface d’attaque. Les composants exposés sont limités au load balancer, tandis que la base de données reste isolée dans des sous-réseaux privés.

---

## IMDSv2

Si possible, garder **IMDSv2 requis** sur l’instance.

### Justification prête

> L’usage d’IMDSv2 renforce la protection des métadonnées d’instance en limitant les abus liés à l’accès aux credentials temporaires via SSRF ou accès indirect aux métadonnées.

---

## Monitoring et alerting

## Minimum crédible à mettre en place

### CloudWatch Alarms

Créer des alarmes défendables comme :

- **EC2 CPUUtilization > 70 %**
- **ALB UnHealthyHostCount >= 1**
- **StatusCheckFailed_Instance >= 1**
- **RDS CPUUtilization > 70 %**
- **RDS FreeStorageSpace trop bas**

### Pourquoi c’est bien

- détecte saturation,
- détecte panne de cible,
- détecte incident sur l’instance,
- détecte pression sur la base.

### Justification prête

> Le monitoring a été renforcé pour détecter les dégradations de disponibilité, les hôtes non sains derrière l’ALB, les incidents EC2 et les tensions sur la base RDS. Cela améliore la réactivité opérationnelle et la fiabilité globale de l’environnement.

---

## Logs et preuves

À capturer si possible :

- cible **healthy** dans le Target Group,
- alarme CloudWatch créée,
- metrics EC2 / ALB / RDS,
- `systemctl status apache2`,
- page UrbanHub fonctionnelle,
- SG EC2 / SG RDS.

---

## Matrice simple de risques

| Risque | Impact | Probabilité | Mesure de réduction |
|---|---:|---:|---|
| Exposition directe de la base | Très fort | Moyen | RDS privée + SG restrictif |
| Exposition directe des EC2 | Fort | Moyen | trafic HTTP uniquement depuis ALB |
| Permissions IAM trop larges | Fort | Moyen | moindre privilège |
| Incident non détecté | Fort | Élevé | CloudWatch alarms |
| Saturation applicative | Moyen | Moyen | ASG + target tracking CPU |
| Panne de cible ALB | Fort | Moyen | health checks + alarme UnHealthyHostCount |

---

## Justifications techniques prêtes à l’emploi

### Pourquoi ALB + Target Group ?

> L’ALB centralise l’exposition HTTP, répartit la charge entre les instances et permet de contrôler leur santé avec des health checks. Cela améliore la disponibilité et évite d’exposer directement chaque instance au trafic Internet.

### Pourquoi ASG ?

> L’Auto Scaling Group permet de maintenir au moins une instance disponible et d’absorber une hausse de charge via une politique de scaling basée sur la CPU. Cela répond aux objectifs de résilience et d’élasticité.

### Pourquoi RDS privée ?

> La base de données contient les données métier et ne doit pas être exposée publiquement. Son placement dans des subnets privés, combiné à un SG restrictif, réduit fortement la surface d’attaque.

### Pourquoi SSH restreint à une IP ?

> Restreindre l’administration SSH à une seule adresse source réduit le risque d’exposition large de l’accès d’administration sur Internet.

### Pourquoi CloudWatch ?

> Le monitoring et les alarmes permettent de passer d’une infrastructure simplement fonctionnelle à une infrastructure administrable, observable et plus robuste en production.

---

## Preuves à capturer

### IAM

- rôle EC2,
- policy attachée,
- absence de droits inutiles visibles.

### Réseau

- SG ALB,
- SG EC2,
- SG RDS,
- subnets publics / privés,
- route table publique.

### Disponibilité

- cible healthy,
- ALB,
- ASG,
- politique de scaling.

### Monitoring

- alarmes CloudWatch,
- graphiques metrics EC2 / ALB / RDS.

### Application

- page UrbanHub,
- `systemctl status apache2`.

---

# Checklist finale

## Avant rendu P1

- [ ] VPC correct
- [ ] 2 subnets publics + 2 privés
- [ ] auto-assign public IPv4 activé sur les subnets publics
- [ ] IGW + route publique OK
- [ ] SG ALB / EC2 / RDS corrects
- [ ] RDS privée
- [ ] rôle EC2 OK
- [ ] Launch Template complet
- [ ] Target Group OK
- [ ] ALB OK
- [ ] ASG OK
- [ ] cible healthy
- [ ] UrbanHub fonctionne
- [ ] captures obligatoires faites

## Avant rendu P2

- [ ] diagnostic des risques rédigé
- [ ] IAM justifié
- [ ] exposition réseau justifiée
- [ ] RDS privée prouvée
- [ ] SSH restreint à votre IP
- [ ] monitoring / alerting visible
- [ ] preuves capturées
- [ ] arguments prêts pour justifier chaque choix

---

# Commandes utiles

## DNS de l’ALB

```bash
aws elbv2 describe-load-balancers --names urbanhub-alb --query "LoadBalancers[0].DNSName" --output text
```

## Apache

```bash
sudo systemctl status apache2
```

## Logs du script d’initialisation

```bash
sudo tail -n 100 /var/log/cloud-init-output.log
```

## Test local depuis l’EC2

```bash
curl -I http://localhost/
curl -I http://localhost/health.php
```

## SSH

```bash
chmod 400 urbanhub-key2.pem
ssh -i "urbanhub-key2.pem" ubuntu@IP_PUBLIQUE
```

---

## Dernier conseil

Pendant l’examen :

- **P1** : suivre l’ordre, ne pas improviser, capturer les preuves au fur et à mesure.
- **P2** : raisonner par couches : **IAM -> réseau -> exposition -> monitoring -> durcissement -> preuves**.

