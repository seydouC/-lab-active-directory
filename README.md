# Étape 2 : service DHCP sur le serveur

## Objectif
Faire distribuer automatiquement les adresses IP aux postes du domaine par le serveur Windows, au lieu de les configurer à la main sur chaque machine.

## Environnement
- Serveur : `SRV-AD-01` (Windows Server 2022), contrôleur du domaine `meyes.local`, adresse IP fixe `192.168.10.100`
- Client : poste Windows 10 joint au domaine (nom d'hôte `DESKTOP-AM8UL1Q`)
- Hyperviseur : VirtualBox

## Démarche
1. Installation du rôle **Serveur DHCP** sur `SRV-AD-01` et autorisation du serveur dans le domaine
2. Création d'une étendue IPv4 `Etendue-Meyes` pour le réseau `192.168.10.0/24`
3. Définition du pool d'adresses distribuées : de `192.168.10.150` à `192.168.10.200`, ce qui laisse les adresses plus basses pour les machines à IP fixe (dont le serveur)
4. Passage du client en configuration automatique (DHCP) et vérification

## Vérifications

### Pool d'adresses
La console DHCP affiche la plage distribuée : `192.168.10.150` à `192.168.10.200`.

![Pool d'adresses](captures/01-pool-adresses.png)

### Baux d'adresses
Le client apparaît dans les baux avec l'adresse `192.168.10.150` (type DHCP). Cette capture date d'avant le renouvellement du bail effectué ensuite sur le client (voir ci-dessous).

![Baux d'adresses](captures/02-baux-adresses.png)

### Configuration reçue par le client
`ipconfig /all` sur le client, après `ipconfig /release` puis `ipconfig /renew` :
- DHCP activé : Oui
- Adresse IPv4 : `192.168.10.150`, masque `255.255.255.0`
- Serveur DHCP : `192.168.10.100`
- Serveur DNS : `192.168.10.100`
- Bail obtenu le 1er novembre 2026, expirant le 9 novembre 2026 (durée de 8 jours)

![ipconfig /all sur le client](captures/03-ipconfig-client.png)

Les GPO configurées précédemment restent appliquées au client après le passage en DHCP.

## Remarque
Une première capture de `ipconfig /all` affichait une date d'obtention du bail en 1890. L'horloge de la VM cliente avait dérivé : le bail a été renouvelé après avoir remis le client à l'heure, et la capture ci-dessus est celle du bail renouvelé. Dans un domaine Active Directory, l'heure des machines doit rester cohérente avec celle du serveur.

## Compétences mises en pratique
Installation et configuration du rôle DHCP, étendue et pool d'adresses, baux, vérification côté client (`ipconfig /all`, `/release`, `/renew`), diagnostic d'un décalage d'horloge.
