# Lab Active Directory — domaine meyes.local

Je suis étudiant en M1 réseau et cybersécurité, et je cherche une alternance dans ce domaine. Pour progresser et avoir quelque chose de concret à montrer en entretien, j'ai monté de A à Z un petit domaine Active Directory chez moi, en VirtualBox. Ce dépôt documente ce que j'ai fait, comment, et ce que j'ai appris en cours de route — y compris les galères, parce que c'est souvent là qu'on apprend le plus.

## Ce que je voulais apprendre

Active Directory est utilisé dans presque toutes les entreprises, et c'est aussi une cible très étudiée en sécurité. Je voulais comprendre comment il fonctionne de l'intérieur avant de m'attaquer à la partie attaque/défense : créer un domaine, organiser des utilisateurs, et surtout vérifier que les stratégies de groupe (GPO) font vraiment ce qu'elles sont censées faire, pas juste les créer et espérer que ça marche.

## Mon environnement

| Élément | Détail |
|---|---|
| Hyperviseur | VirtualBox |
| Contrôleur de domaine | Windows Server 2022 — `SRV-AD-01` |
| Deuxième contrôleur de domaine | Windows Server 2022 — `SRV-AD-02`, 192.168.10.101 |
| Client | Windows 10 Pro — `CLIENT-WINDOWS10` |
| Domaine | `meyes.local` |
| IP du serveur | 192.168.10.100 (DNS configuré sur lui-même) |
| IP du client | 192.168.10.150 (attribuée par DHCP, étendue .150 à .200) |
| Réseau | Réseau interne VirtualBox, les deux VM isolées sur le même segment |

## Schéma

```
Réseau interne VirtualBox (192.168.10.0/24)

[ SRV-AD-01 ]  <----->  [ CLIENT-WINDOWS10 ]
Windows Server 2022      Windows 10
Controleur de domaine    Membre du domaine
192.168.10.100            192.168.10.150
Domaine : meyes.local
```


## Ce que j'ai mis en place

### Les OU et les utilisateurs

J'ai imaginé une petite entreprise fictive avec quatre services, et un utilisateur type par service :

| Service (OU) | Utilisateur | Identifiant |
|---|---|---|
| DIRECTION | Rokia Coulibaly | rokia.coulibaly |
| LOGISTIQUE | Gaoussou Coulibaly | gaoussou.coulibaly |
| COMPTABILITE | Ibrahim Coulibaly | ibrahim.coulibaly |
| INFORMATIQUE | Mamadou Coulibaly | mamadou.coulibaly |

### Les GPO, et surtout : est-ce qu'elles marchent vraiment ?

Créer une GPO, c'est facile. Je voulais aller plus loin et vérifier concrètement qu'elle s'applique, pas juste qu'elle existe. Pour chacune, j'ai suivi la même méthode :
1. Connexion sur le client avec le compte concerné.
2. `gpupdate /force` pour forcer l'application.
3. `gpresult /r /scope user` pour confirmer qu'elle apparaît bien dans les stratégies appliquées.
4. Un test réel : j'essaie d'ouvrir ce qui doit être bloqué.

| GPO | Service visé | Ce qu'elle bloque | Résultat |
|---|---|---|---|
| `INTERDIRE-PANNEAU-CONF` | COMPTABILITE | Le Panneau de configuration | ✅ Bloqué, avec le message « Cette opération a été annulée en raison de restrictions sur cet ordinateur » |
| `RESTRICTION-CMD` | LOGISTIQUE | L'invite de commandes | ✅ Bloquée, avec le message « L'invite de commandes a été désactivée par votre administrateur » |

### Le service DHCP

Le serveur distribue maintenant les adresses du réseau : étendue `Etendue-Meyes`, de `192.168.10.150` à `192.168.10.200`. Le client reçoit son adresse automatiquement (`192.168.10.150`), avec un bail de 8 jours. [Voir le détail et les captures](02-dhcp/README.md)

### Les GPO de sécurité

Politique de mot de passe sur tout le domaine : 12 caractères minimum, complexité activée, durée de vie de 90 jours. Verrouillage du compte après 5 tentatives, pour 30 minutes. J'ai vérifié sur le client qu'un compte se verrouille bien après des mots de passe incorrects. [Voir le détail et les captures](03-gpo-securite/README.md)

### Le dossier partagé par service

Un partage `\\SRV-AD-01\partage` avec un sous-dossier par service. Chaque service a son groupe de sécurité (`GG-COMPTABILITE`, `GG-LOGISTIQUE`, `GG-DIRECTION`, `GG-INFORMATIQUE`) avec le droit Modification sur son seul dossier. J'ai testé avec les quatre utilisateurs : chacun ouvre son dossier et est refusé sur les autres. [Voir le détail et les captures](04-dossier-partage/README.md)

### Le deuxième contrôleur de domaine

J'ai ajouté un second contrôleur (`SRV-AD-02`) au domaine : l'annuaire a été répliqué depuis `SRV-AD-01`, et `repadmin /replsummary` n'indique aucun échec. J'ai testé la réplication dans les deux sens en créant un utilisateur sur chaque serveur et en vérifiant qu'il apparaissait sur l'autre. [Voir le détail et les captures](05-deuxieme-controleur/README.md)

## Les galères, et comment je m'en suis sorti

Ce lab n'a pas été une ligne droite, et je trouve que c'est important de le montrer :

- **Pendant plusieurs jours, impossible de me connecter au serveur.** Windows me répondait « mot de passe incorrect » alors que j'étais sûr de mon mot de passe. J'ai testé plusieurs pistes (disposition du clavier dans la VM, réseau, snapshot), sans trouver la cause exacte. J'ai essayé de réinitialiser le mot de passe via le mode de restauration des services d'annuaire (DSRM) en démarrant sur l'ISO d'installation, mais plutôt que d'insister, j'ai choisi de recréer le contrôleur de domaine avec les mêmes OU et les mêmes GPO, pour avancer plus vite sur la partie que je voulais vraiment pratiquer : la gestion des GPO.
- **Le client refusait de rejoindre le domaine** avec l'erreur « le nom réseau spécifié n'est plus disponible ». En vérifiant, le serveur n'avait pas encore fini de démarrer tous ses services au moment de la tentative. Il a suffi d'attendre quelques minutes et de réessayer.
- **Un utilisateur du domaine n'arrivait pas à se connecter**, malgré le bon mot de passe. En fait, je tapais son prénom au lieu de son véritable identifiant de connexion (`ibrahim` au lieu de `ibrahim.coulibaly`). `Get-ADUser -Filter * | Select Name, SamAccountName` m'a permis de vérifier les vrais identifiants.
- **Le client perdait l'accès au partage, et son horloge était décalée.** Pendant les tests du dossier partagé, le client perdait par moments l'accès à `\\SRV-AD-01\partage`. Son heure était en retard sur celle du serveur (au point d'afficher un bail DHCP daté de 1890), et `w32tm /resync` échouait. Le service de temps du serveur tournait, mais sa source était sa propre horloge (`Local CMOS Clock`), faute de source externe dans le lab. J'ai réglé l'heure du client à la main, renouvelé le bail avec `ipconfig /release` puis `/renew`, et les coupures ont cessé. Dans un domaine, un écart de plus de 5 minutes entre deux machines peut gêner l'authentification, ce qui est probablement lié. [Détail dans le rapport du dossier partagé](04-dossier-partage/README.md)

## Et après ?

Ce lab continue d'évoluer, je mets ce dépôt à jour au fur et à mesure :

- [x] Mettre en place le DHCP sur le serveur
- [x] Ajouter des GPO de sécurité (complexité des mots de passe, verrouillage de compte)
- [x] Créer un dossier partagé avec des permissions différentes selon les services
- [x] Ajouter un deuxième contrôleur de domaine et vérifier la réplication
- [ ] Mettre en place la sauvegarde et la restauration
- [ ] Simuler des pannes et les dépanner
- [ ] Ajouter une VM Kali Linux isolée pour observer comment un domaine AD peut être énuméré (BloodHound)
- [ ] Suivre des parcours TryHackMe (Pre Security, puis Jr Penetration Tester ou SOC Level 1)
- [ ] Automatiser certaines tâches avec PowerShell
- [ ] Tester un environnement hybride avec Entra ID

## Compétences pratiquées

`Active Directory` `Windows Server` `GPO` `DNS` `DHCP` `Groupes de sécurité` `Permissions NTFS` `Contrôleur de domaine` `Réplication Active Directory` `VirtualBox` `Administration réseau` `Dépannage système`
