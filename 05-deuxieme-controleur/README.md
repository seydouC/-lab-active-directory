# Étape 5 : deuxième contrôleur de domaine et réplication

## Objectif
Ajouter un deuxième contrôleur de domaine (`SRV-AD-02`) au domaine `meyes.local`, pour que l'annuaire existe sur deux serveurs et se réplique entre eux, au lieu de dépendre d'un seul serveur.

## Environnement
- Premier contrôleur : `SRV-AD-01` (Windows Server 2022), IP fixe `192.168.10.100`
- Deuxième contrôleur : `SRV-AD-02` (Windows Server 2022), IP fixe `192.168.10.101`
- Réseau interne VirtualBox (`intnet`), les machines sont isolées sur le même segment
- Hyperviseur : VirtualBox

## Démarche
1. Création d'une VM neuve `SRV-AD-02` (2 Go de mémoire, 2 processeurs, disque de 50 Go) et installation de Windows Server 2022 avec interface graphique, de la même version que le premier serveur
2. Adresse IP fixe `192.168.10.101`, masque `255.255.255.0`, serveur DNS préféré : `192.168.10.100` (le premier contrôleur), sans passerelle
3. Vérification que le domaine est trouvé : `nslookup meyes.local 192.168.10.100`
4. Renommage de la machine en `SRV-AD-02` et jonction au domaine `meyes.local`
5. Installation du rôle **Services AD DS**
6. Promotion en contrôleur de domaine : « Ajouter un contrôleur de domaine à un domaine existant », avec serveur DNS et catalogue global, réplication depuis `SRV-AD-01`
7. Redémarrage automatique, puis ouverture de session avec un compte du domaine

![nslookup depuis SRV-AD-02](05-01-nslookup.png)

![Assistant de configuration des services de domaine](05-02-assistant-deploiement.png)

La vérification de la configuration requise s'est terminée sans erreur. L'avertissement sur la délégation DNS est attendu dans un domaine en `.local` (pas de zone parente) et n'appelle aucune action.

![Vérification de la configuration requise](05-03-verification-configuration.png)

## Vérifications

### L'annuaire a été copié
Sur `SRV-AD-02`, les OU COMPTABILITE, DIRECTION, INFORMATIQUE et LOGISTIQUE sont présentes alors qu'elles n'ont été créées que sur `SRV-AD-01` : l'annuaire a été répliqué.

![Annuaire vu depuis SRV-AD-02](05-04-annuaire-sur-srv-ad-02.png)

### Les deux contrôleurs sont déclarés
L'OU **Domain Controllers** contient `SRV-AD-01` et `SRV-AD-02`, tous deux catalogues globaux, dans le même site.

![Domain Controllers](05-05-domain-controllers.png)

### État de la réplication
`repadmin /replsummary` n'indique aucun échec dans les deux sens (0 sur 5 pour chaque contrôleur).

![repadmin /replsummary](05-06-repadmin-replsummary.png)

### Test dans les deux sens
- Un utilisateur `Test Replication` créé sur `SRV-AD-01` apparaît sur `SRV-AD-02`
- Un utilisateur `Test Retour` créé sur `SRV-AD-02` apparaît sur `SRV-AD-01`

![Test Replication visible sur SRV-AD-02](05-07-test-replication-sur-srv-ad-02.png)

![Test Retour visible sur SRV-AD-01](05-08-test-retour-sur-srv-ad-01.png)

## Problèmes rencontrés
- **Identifiants refusés dans l'assistant de promotion** : l'assistant utilisait le compte administrateur local de la machine. Il faut un compte du domaine : `Administrateur@meyes.local`. Une première saisie avec une barre oblique normale (`MEYES/Administrateur`) donnait aussi un nom de compte invalide.
- **Machine très lente après la promotion** : avec 2 Go de mémoire, un contrôleur de domaine fraîchement promu est lent pendant plusieurs minutes. J'ai attendu au lieu de forcer l'arrêt, ce qui aurait pu abîmer l'annuaire.

## Pas encore testé
La tolérance de panne (éteindre un contrôleur et vérifier que le domaine continue de fonctionner) fait partie de la suite du lab.

## Compétences mises en pratique
Installation du rôle AD DS, promotion d'un contrôleur de domaine supplémentaire, DNS intégré à l'annuaire, catalogue global, réplication Active Directory, vérification avec `repadmin`, diagnostic d'erreurs d'authentification.
