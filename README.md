EC04 - UrbanHub Park
Guide de révision et de secours pour l'examen
Partie 1 (implémentation) + Partie 2 (sécurisation)
But du document
Vous aider a refaire rapidement la P1, puis a attaquer la P2 avec une methode claire, des
controles prioritaires, des captures utiles et des justifications techniques simples.
Ce guide est volontairement pratique: ordre d'execution, verifications de fin d'etape, depannage
rapide, et liste de preuves a produire.
Document Usage conseille pendant l'examen
P1 Suivre la checklist, ne pas improviser l'ordre,
capturer les preuves au fur et a mesure.
P2
Partir d'un diagnostic par couches: IAM ->
reseau -> exposition -> monitoring ->
durcissement -> preuves.
EC04 UrbanHub - Guide examen
1. Partie 1 - Refaire UrbanHub rapidement et proprement
Objectif. Monter une architecture AWS fonctionnelle dans eu-west-3: EC2 derriere un ALB,
instances gerees par un Auto Scaling Group, base MariaDB sur RDS non exposee publiquement,
routage correct, script d'installation lance au boot.
Rappel des exigences du sujet P1
• Region unique: eu-west-3.
• EC2 derriere un Application Load Balancer avec un target group HTTP:80.
• Auto Scaling Group: min 1, max 8, desired 1, cible CPU 50 %.
• Base MariaDB sur Amazon RDS, non accessible directement depuis Internet.
• VPC 10.0.0.0/16 avec 2 subnets publics (10.0.1.0/24, 10.0.2.0/24) et 2 subnets prives (10.0.3.0/24,
10.0.4.0/24).
• Subnets publics: auto-assign public IPv4 active.
• Role EC2 avec AmazonRDSReadOnlyAccess pour le script d'installation.
Ordre ideal d'execution P1 (a suivre strictement)
Ordre Action Controle de fin
1
VPC + 4 subnets + DNS + IGW +
route table publique
Les 2 subnets publics ont
0.0.0.0/0 vers l'IGW et autoassign public IPv4 active.
2
Security groups ALB / EC2 /
RDS
ALB:80 depuis Internet,
EC2:80 depuis SG ALB, EC2:22
depuis votre IP, RDS:3306
depuis SG EC2.
3
RDS MariaDB + DB subnet
group prive
Public access = No, DB subnet
group avec les 2 subnets
prives.
4 Role IAM EC2
Role attache, policy
AmazonRDSReadOnlyAccess
visible.
5 Launch Template AMI Ubuntu, key pair, SG EC2,
role IAM, user data complet.
6 Target group + health check HTTP:80, path /health.php,
code 200.
7 ALB
Internet-facing, 2 subnets
publics, listener HTTP:80 ->
target group.
8 ASG
Min 1, desired 1, max 8, target
tracking CPU 50 %, rattache au
target group.
EC04 UrbanHub - Guide examen
P1 - Etape 1: reseau
• Creer le VPC 10.0.0.0/16 et activer DNS resolution + DNS hostnames.
• Creer les 4 subnets dans 2 AZ. Publics: 10.0.1.0/24 et 10.0.2.0/24. Prives: 10.0.3.0/24 et 10.0.4.0/24.
• Creer une Internet Gateway et l'attacher au VPC.
• Creer une route table publique avec 0.0.0.0/0 -> IGW, l'associer aux 2 subnets publics.
• Laisser les subnets prives sur une table privee sans route Internet.
• Verifier que les 2 subnets publics ont l'attribut auto-assign public IPv4 active.
P1 - Etape 2: groupes de securite
• SG ALB: inbound HTTP 80 depuis 0.0.0.0/0, outbound tout.
• SG EC2: inbound HTTP 80 depuis SG ALB; inbound SSH 22 depuis votre IP; outbound tout.
• SG RDS: inbound 3306 depuis SG EC2 uniquement.
P1 - Etape 3: base RDS
• Creer un DB subnet group avec les 2 subnets prives.
• Creer MariaDB RDS dans ce DB subnet group.
• Laisser Public access = No.
• Conserver les identifiants qui seront reutilises dans le user data du Launch Template.
P1 - Etape 4: role IAM EC2
• Creer un role de confiance EC2.
• Attacher AmazonRDSReadOnlyAccess.
• Verifier que le role apparait ensuite sur l'instance lancee par l'ASG.
P1 - Etape 5: Launch Template
• Choisir Ubuntu Server.
• Ajouter la key pair.
• Ne pas fixer le subnet dans le template; laisser l'ASG choisir les subnets publics.
• Associer le SG EC2 et le role IAM.
• Coller le user data complet.
User data type a conserver dans le Launch Template
#!/bin/bash
export MASTER_DB_USER=admin
export MASTER_DB_PASSWORD='SuperPassword123!'
export AWS_REGION='eu-west-3'
curl -fsSL https://raw.githubusercontent.com/dahut87/urban_1/refs/heads/main/
install.sh | bash
EC04 UrbanHub - Guide examen
P1 - Etapes 6 a 8: ALB, Target Group, ASG
• Target group: type Instance, HTTP:80, path /health.php, code 200.
• ALB: internet-facing, 2 subnets publics, listener HTTP:80 qui transfere vers le target group.
• ASG: min 1, desired 1, max 8, rattache au target group, 2 subnets publics.
• Politique de scaling: target tracking sur Average CPU Utilization = 50 %.
• Verifier que le target group n'est pas vide et qu'une cible passe en healthy.
P1 - Pannes les plus frequentes et correction express
Symptome Cause probable Correction
502 / 503 sur le DNS ALB Target group vide ou instance
unhealthy
Verifier que l'ASG est bien
rattache au target group, puis
attendre une cible healthy.
Target group: 0 cible ASG non rattache ou aucune
instance
Reattacher l'ASG au target
group, desired=1, verifier les
subnets publics.
Pas de SSH Pas de key pair ou pas d'IP
publique
Nouvelle version du Launch
Template avec key pair,
verifier auto-assign public
IPv4 sur les subnets publics.
RDS inaccessible SG RDS faux ou RDS publique /
mauvais subnet group
Laisser Public access = No, SG
RDS ouvert a 3306 seulement
depuis SG EC2.
Health check KO Apache / PHP / user data pas
termine ou path faux
Verifier cloud-init logs,
apache2, et path /health.php.
Captures minimales a rendre pour la P1
• Application UrbanHub fonctionnelle via le DNS de l'ALB.
• Commande CloudShell qui affiche le DNS de l'ALB + resultat.
• systemctl status apache2 sur une instance EC2.
• En bonus: target group healthy, ASG min/desired/max, RDS Public access = No.
EC04 UrbanHub - Guide examen
2. Partie 2 - Securisation de l'environnement Cloud
Ce que dit l'enonce. Analyser, corriger et prevenir les vulnerabilites de l'environnement cree en
P1. Contenu attendu: diagnostic des risques Cloud, gestion IAM, bonnes pratiques provider,
monitoring/alerting, durcissement. Livrable: environnement securise + justification technique +
elements de preuve.
Objectif reel de la P2
• Montrer que vous savez identifier ce qui expose l'infrastructure inutilement.
• Corriger les permissions trop larges, les ouvertures reseau inutiles et les ressources mal
exposees.
• Ajouter un minimum de supervision utile et defendable.
• Expliquer pourquoi chaque correction reduit le risque sans casser la P1.
Methode P2 en 7 passes (ordre conseille)
Pass Question a se poser Action attendue
1. Exposition Qu'est-ce qui est visible depuis
Internet ?
Verifier ALB public, EC2
limitee, RDS non publique,
SSH restreint.
2. IAM Qui a le droit de faire quoi ?
Examiner les roles, policies et
supprimer le trop large si
possible.
3. Reseau Les flux sont-ils limites au
necessaire ?
Revoir les SG et, si utile, les
route tables.
4. Donnees / metadata Qu'est-ce qui peut fuiter ? Verifier IMDSv2, acces
metadata, base non publique.
5. Monitoring Comment detecter une panne
ou un abus ?
Ajouter des alarmes
CloudWatch simples et utiles.
6. Durcissement
Quels reglages renforcent la
posture sans casser le
service ?
Tighten SSH, eventuellement
HTTPS/443 si demande, traces
et logs.
7. Preuves
Comment prouver chaque
mesure ?
Captures ecran, tableaux
avant/apres, justification
technique courte.
Matrice de risques UrbanHub P2
Ressource Risque Correction prioritaire Preuve utile
ALB
Exposition web
normale mais
surveillee trop
faiblement
Verifier listener,
target group sain,
eventuellement
CloudWatch sur
UnHealthyHostCount
Capture ALB +
alarmes
EC2 SSH trop ouvert / SSH seulement depuis Capture SG EC2
EC04 UrbanHub - Guide examen
acces direct web
inutile
votre IP; HTTP
seulement depuis SG
ALB
RDS
Base publiquement
accessible ou flux trop
larges
Public access = No; SG
3306 seulement
depuis SG EC2
Capture RDS + SG RDS
IAM role EC2 Permissions trop
larges
Conserver seulement
le necessaire au
fonctionnement
Capture role + policies
Subnets
EC2 sans IP publique
ou mauvaise
exposition
Public subnets bien
configures; DB en
private subnets
Capture subnets +
route tables
Metadonnees EC2
Abus IMDSv1 ou
metadata trop
permissives
Exiger IMDSv2 Capture options
metadata
Supervision Panne non detectee a
temps
Alarmes CloudWatch
simples sur CPU /
status / target health
Capture alarmes
P2 - Axe 1: IAM et permissions
• Commencer par l'inventaire des roles utilises: role EC2, eventuelles policies inline, comptes IAM
manipules pendant le TP.
• Question cle: la permission est-elle strictement utile au service ? Si non, la resserrer.
• Pour UrbanHub, le point defendable est de garder le role EC2 necessaire au script d'installation
et d'eviter d'ajouter des droits larges sans justification.
• En justification: le principe du moindre privilege reduit l'impact d'une compromission
d'instance.
P2 - Axe 2: reseau et exposition
• ALB public: normal, car l'application doit etre accessible depuis Internet.
• EC2: ne pas exposer HTTP 80 au monde; n'autoriser le port 80 qu'a partir du SG de l'ALB.
• SSH: n'ouvrir qu'a votre IP, et si l'examen le permet, supprimer ou resserrer la regle apres debug.
• RDS: Public access = No et SG 3306 seulement depuis le SG EC2.
• En justification: reduction de la surface d'attaque et segmentation des flux par couche.
P2 - Axe 3: monitoring et alerting
• Ajouter au minimum des alarmes CloudWatch simples, lisibles et defendables.
• Exemples defendables: CPU EC2 elevee, StatusCheckFailed pour l'instance, UnHealthyHostCount
pour le target group, CPU ou FreeStorageSpace pour RDS si disponible.
EC04 UrbanHub - Guide examen
• Le but n'est pas d'en mettre 20, mais de montrer que vous savez detecter indisponibilite et
degradation.
• En justification: meilleure detection des incidents et reactivite operationnelle.
P2 - Axe 4: durcissement
• Verifier que l'instance exige IMDSv2. C'est une mesure simple et robuste contre certains abus de
metadata.
• Verifier que la base reste privee et que les flux sont minimaux.
• Verifier les options de lancement et les composants critiques: key pair, SG, role IAM, user data
maitrise.
• Si l'exercice vous pousse vers le HTTPS, vous pouvez expliquer qu'un vrai durcissement web
passe par TLS sur l'ALB avec certificat ACM; ne le faites que si le sujet l'oriente clairement.
Alarmes CloudWatch defendables pour la P2
Alarme Seuil simple Interet Preuve a garder
EC2 CPUUtilization >= 80 % pendant 5
min
Detecter surcharge ou
runaway process
Capture de l'alarme +
etat OK/ALARM
EC2
StatusCheckFailed >= 1
Detecter incident
machine /
hyperviseur / reseau
Capture alarme
ALB
UnHealthyHostCount >= 1 Detecter perte de cible
saine
Capture metrique ou
alarme
RDS CPUUtilization >= 80 % Detecter saturation
DB
Capture alarme
RDS FreeStorageSpace Sous un seuil prudent
Detecter risque de
panne par manque
d'espace
Capture alarme
Note - Adaptez le jeu d'alarmes a ce que la console vous laisse creer rapidement. Le bon reflexe est
de privilegier 3 a 5 alarmes pertinentes plutot qu'une collection confuse.
Justifications techniques courtes a reutiliser
Mesure Formulation simple et defendable
SSH restreint a mon IP Je reduis la surface d'attaque en limitant
l'administration a une source connue.
EC2 accessible seulement via ALB Je force le passage par la couche de repartition
et j'evite une exposition directe des instances.
RDS non publique
Je protege la base en la gardant dans des
subnets prives et en filtrant le flux 3306 au
strict necessaire.
Role IAM limite J'applique le moindre privilege pour limiter
l'impact d'un compte ou d'une instance
EC04 UrbanHub - Guide examen
compromise.
IMDSv2 required Je renforce l'acces aux metadonnees d'instance
en imposant le token IMDSv2.
Alarmes CloudWatch
Je rends la plateforme plus exploitable en
detectant plus vite une panne ou une
surcharge.
P2 - Check-list de verification finale
□ RDS: Public access = No.
□ SG EC2: port 80 autorise seulement depuis SG ALB.
□ SG EC2: port 22 limite a votre IP ou retire apres validation si le sujet le permet.
□ SG RDS: 3306 seulement depuis SG EC2.
□ Role IAM verifie et policies defendables.
□ IMDSv2 required sur l'instance si possible.
□ Au moins 3 alarmes CloudWatch pertinentes creees.
□ Target group avec cible saine, ALB fonctionnel, application toujours accessible apres
durcissement.
EC04 UrbanHub - Guide examen
3. Preuves, commandes et depannage rapide
Captures P2 a privilegier
Theme Capture utile
IAM Role EC2 + policies visibles
SG EC2 Regles 80 depuis SG ALB et 22 depuis votre IP
SG RDS Regle 3306 depuis SG EC2 uniquement
RDS Public access = No
Monitoring Liste des alarmes CloudWatch creees
IMDS Options metadata avec IMDSv2 required si
active
Service final Application toujours fonctionnelle apres
durcissement
Commandes utiles pendant l'examen
Usage Commande
CloudShell - Recuperer le DNS ALB
aws elbv2 describe-load-balancers --names
urbanhub-alb --query
"LoadBalancers[0].DNSName" --output text
SSH - Etat Apache sudo systemctl status apache2
SSH - Logs cloud-init sudo tail -n 100 /var/log/cloud-initoutput.log
SSH - Test local app curl -I http://localhost/
SSH - Test health check curl -I http://localhost/health.php
Diagnostic express si ca casse apres un durcissement
Symptome Reflexe
ALB ne repond plus Verifier target group, SG EC2, health check
/health.php, apache2.
SSH perdu Verifier IP source du port 22, IP publique, key
pair, instance remplacee par l'ASG.
Base inaccessible Verifier SG RDS, endpoint DB, Public access,
role IAM si le script s'appuie dessus.
Application en erreur mais cible healthy Regarder logs Apache / PHP et tester localhost
sur l'instance.
Apres changement ASG, plus de cible Verifier que l'ASG est toujours rattache au bon
target group.
4. References AWS officielles a connaitre
• EC2 user data - execution de scripts au lancement.
EC04 UrbanHub - Guide examen
• Application Load Balancer target group health checks.
• EC2 Auto Scaling target tracking policies.
• CloudWatch alarms - alarmes CPU / statut.
• Security groups for EC2 / VPC security groups.
• Subnet auto-assign public IPv4.
• RDS public vs private access et DB subnet groups.
• IMDSv2 pour les metadonnees d'instance.
Dernier conseil pour l'examen
• En P2, ne cherchez pas a tout faire. Cherchez a montrer une methode, des corrections
pertinentes et des preuves propres.
• Chaque modification doit pouvoir se justifier en une phrase: quel risque je reduis ? pourquoi
ma correction est adaptee ? comment je prouve qu'elle est en place ?
• Ne cassez pas la P1 en voulant sur-securiser. Le bon rendu P2 est un environnement encore
fonctionnel, mais mieux protege et mieux surveille.
