test.html.md

# Mise en place d’un réseau d’entreprise

## Introduction

L’objectif de ce projet est de disposer d’une ébauche de réseau tel qu’on pourrait trouver dans une entreprise, avec une DMZ contenant un serveur proposant plusieurs services (pour l’instant juste une instance apache2) et un réseau privé contenant les machines clientes de l’entreprise.

La totalité du réseau tourne sur des VMs Debian Bullseye 11.1 virtualisées sur VirtualBox.

## Sommaire

-   [Détails des outils utilisés](#d%C3%A9tails)
-   [Plan d’adressage](#plan_d'adressage)
-   [Interconnexion de la DMZ et du réseau privé](#)
-   [Interconnexion avec un réseau identique géré par une autre personne](#)
-   [Connexion des réseaux privés à Internet via le routeur](#)
-   [Mise en place d’une politique de sécurité](#)
-   [DHCP](#dhcp)
-   [Proxy](#proxy)
-   [DNS](#dns)

## Détails des outils utilisés

-   [Serveur Web : apache2](#apache2)
-   Navigateur web : Firefox
-   [Proxy : squid](#squid)
-   [Serveur DHCP : isc-dhcp-server](#dhcp)
-   [DNS : bind](#dns)

## Plan d’adressage

Le serveur est situé sur le sous-réseau **DMZ** _**192.168.1.0/24**_ (adresse IP fixé à XXX.1.1 via la configuration DHCP)  
Les clients sont situés sur le sous-réseau **private** _**192.168.2.0/24**_  
Le routeur fera office de serveur DHCP, de proxy et de serveur DNS. Ses adresses sont _**192.168.1.1**_ (private) et _**192.168.2.1**_ (DMZ). Son adresse publique est _**10.10.2.16**_ (_gérée par VirtualBox via une connexion NAT_).
Le routeur dispose également d'un connexion par pont.

## apache2

Afin de pouvoir tester la communication client - serveur, il a été décidé de mettre en place un serveur apache2 hebergé sur le serveur et proposant un site web.  
La configuration est celle par défaut, le besoin de disposer de plusieurs serveurs en parallèle ne s’exprimant pas.

## Interconnexion de la DMZ et du réseau privé

Pour que le client puisse accéder au serveur, il faut mettre en place le routage.  
Pour ce faire, il faut premièrement indiquer à la machine routeur de _forward_ les paquets.

```bash
root@routeur:~$	 echo 1 > /proc/sys/net/ipv4/ip_forward


```

Il faut ensuite fournir aux machines clientes et serveur la route à suivre pour accéder l’une à l’autre.

```bash
root@client:~$	 ip route add 192.168.1.2/24 via 192.168.1.1


```

Mais la configuration la plus intelligente des routes est de définir le routeur comme étant la route par défaut des machines.

```bash
root@client:~$	 ip route add default via 192.168.1.1


```

Désormais, un client peut accéder à la page web hébergé par le serveur.  
Il n’y a pas besoin de mettre en place du NAT car les échanges sont routés entre deux réseaux privés (192.168.X.X) et ne seront pas rejetés par le routeur.

## Interconnexion avec un réseau identique géré par une autre personne

Cette partie a été réalisée avec l’aimable collaboration de M. Emrick PESCE.  
Le réseau géré par Emrick est identique au mien (au plan d’adressage près).

### Mise en place du DNAT

Afin de pouvoir requêter le serveur d’Emrick, il faut que mon client connaisse ait accès à son serveur.  
Or la configuration actuelle de nos routeurs rend cela impossible, car le client A a tout au plus accès au routeur B.  
Il faut donc indiquer au routeur que toute requête sur le port 80 (web) afin de rediriger vers le serveur. Cela est totalement transparent pour le client.

```bash
root@client:~$	 iptables -t nat -A PREROUTING -p tcp -d 10.0.2.66 --dport 80 --to 192.168.1.2:80


```

### Branchement au switch

Grace à la contribution de Mme Anaïs THILLAYE, nous disposons d’un switch (Cisco?) sur lequel nous pouvons brancher nos deux machines hôtes en Ethernet en ajoutant une connexion **par pont** sur nos routeurs.

![](https://github.com/MarekPrim/rsex/blob/main/unknown.png)

## Connexion des réseaux privés à Internet via le routeur

Pour l’instant, les machines clientes et serveurs sont connectées à Internet via une interface NAT dans le souci de pouvoir installer des paquets tels que apache2 ou curl.  
Mais cette configuration n’offre presque aucun contrôle de l’entreprise sur le trafic des machines clientes (ils peuvent par exemple accéder à des sites potentiellement infectés).  
Il faut donc faire en sorte que le trafic sortant du réseau privé soit obligé de transiter via le routeur.  
Or, le routeur ne peut “_forward_” le trafic provenant du réseau privé sur Internet car les paquets provenant d’adresses privées doivent être rejetés par les routeurs n’ayant pas de connexion directe dans ce réseau. Les paquets seraient alors directement drop par le prochain routeur.  
Afin de pouvoir faire en sorte que les clients puissent communiquer sur Internet via le routeur, il faut mettre en place du **Source Nating** (_**SNAT**_).

Pour ce faire, il faut d’abord indiquer au routeur qu’il peut remplacer les adresses IPs sources par la sienne.

```bash
root@routeur:~$	 iptables -t nat -A POSTROUTING -o enp0s10 -j MASQUERADE


```

Il faut ensuite ensuite indiquer au routeur quelles adresses IP il peut remplacer.

```bash
root@routeur:~$	 iptables -t nat -A POSTROUTING -o enp0s10 -j SNAT --to 10.0.2.55


```

Les machines clientes ont maintenant accès à Internet via le routeur.

## DHCP

### Configuration du serveur DHCP

Afin d’éviter la configuration manuelle des machines clientes, il faut fournir un service DHCP (_Dynamic Host Configuration Protocol_).  
L’outil choisi pour effectuer ceci est **isc-dhcp-server**, qui remplace dhcp3-server.

Le serveur DNS choisi est le DNS de Cloudflare (1.1.1.1 / 1.0.0.1).

Il faut déclarer dans le fichier **/etc/dhcp/dhcpd.conf** la configuration suivante :

```
subnet 192.168.2.0 netmask 255.255.255.0 {
	range 192.168.2.2 192.168.2.30; #Plages d'adresses pouvant être données à un client
	options routers 192.168.1.1; #Route par défaut pointant sur le routeur
}


```

Pour donner une adresse constante au serveur, on peut l’indentifier via son adresse MAC :

```
host server {
	hardware ethernet 08:00:27:13:70:bb; #Adresse MAC de l'interface du serveur
	fixed-adress 192.168.1.2;


```

Il faut également indiquer à isc-dhcp-server quelles interfaces il doit écouter.  
Il faut pour cela modifier le fichier **/etc/default/isc-dhcp-server** en rajoutant les interfaces à écouter (par défaut, l’interface écoutée est eth0) :

```
INTERFACES="enp0s8 enp0s9"

```

### Configuration des clients

Pour que les machines clientes puissent demander et recevoir des adresses via le serveur DHCP, il faut modifier la configuration de leur interface :

Fichier de configuration **/etc/network/interfaces** :

```
iface enp0s8 inet dhcp

```

### Problèmes rencontrées

Dans certains cas, lors du démarrage des VMs, l'adresse accordée aux machines est ***169.254.8.92/16*** .
Après recherche, cela ne se produit que lors du démarrage d'un VM **cliente** ou **serveur** avant démarrage de la machine **routeur**. En effet, le serveur DHCP n'étant pas encore en cours, les machines comptant sur le DHCP pour récupérer des adresses ne peuvent pas en obtenir et vont s'attribuer une adresse **169.254.X.X** indiquant que le serveur DHCP n'est pas atteignable.

## Proxy

### Proxy cache classique

#### Configuration du proxy

Afin de pouvoir éviter une surcharge inutile du réseau si plusieurs clients souhaitent accéder à la même ressource (la bande passante externe est bien plus coûteuse que la bande passante interne), on peut mettre en place un proxy.  
L’outil utilisé est squid, un proxy cache http/https/ftp combiné à un accélérateur web (ressource compressor).  
Squid est hebergé sur le routeur, et est configuré de tel sorte qu’il n’autorise pas le protocole SSH ni l’accès au site internet [facebook.com](http://facebook.com) .  
Fichier de configuration **/etc/squid/squid.conf** :

```
 http_port 3128 #Configuration par défaut du port
 acl localnet src 192.168.2.0.24 #private network
 acl SSH port 22
 acl Safe port 80
 acl Safe port 443
 acl Safe port 21
 acl Facebook dstdomain dstdomain .facebook.com
 //////
 http_access deny SSH
 http_deny !Safe
 http_deny Facebook


```

#### Configuration du navigateur

Pour que le navigateur utilise le proxy, il faut lui indiquer qu’il doit passer par un proxy :  
![enter image description here](https://www.numelion.com/wp-content/uploads/2017/08/parametrer-proxy-mozilla-firefox.jpg)

### Proxy transparent

Afin de faciliter l’accès à Internet des machines clientes, on peut indiquer au routeur qu’il doit rediriger tout le trafic http sur le proxy afin que le client n’ai pas à configurer son proxy.  
Il s’agit d’une simple redirection des ports 80 et 443 :

```bash
root@routeur:~$ iptables -t nat -A PREROUTING -i enp0s8 -p tcp --dport 80 -j REDIRECT --to-port 3128

root@routeur:~$ iptables -t nat -A PREROUTING -i enp0s8 -p tcp --dport 443 -j REDIRECT --to-port 3128


```

Désormais, les machines clientes n’ont plus à configurer leur navigateur pour utiliser le proxy.

## DNS

L’outil choisi pour réaliser le serveur DNS est **bind9**.
Pour  créer un nom de domaine, il faut créer un nouveau fichier db.nom_de_zone dans le répertoir **/etc/bind/** du serveur.
J'ai fait le choix de créer un nom de domaine correspondant à ma DMZ, mydmz.lan.
Fichier de configuration **db.mydmz.lan**:
```
$TTL    604800
@       IN      SOA     mydmz.lan. webmaster.mail.net. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative 
@       IN      NS      mydmz.lan.
NS      IN      A       192.168.1.2
box     IN      AAAA		::
```

Il faut ensuite déclarer la zone dans le fichier **/etc/bind/named.conf.local** de configuration local de bind9 :

```
zone "mydmz.lan" {
	type master;
	file "/etc/bind/db.mydmz.lan";
}
```

On peut maintenant relancer le service named (aliasé par bind9) et tenter d'accéder au nom de domaine.
Le résultat d'un commande dig sur le nom de domaine retourne une réponse **mais** provenant de 1.0.0.1, le DNS passé en configuration par défaut via le DHCP.
Il faut donc indiquer à notre serveur DHCP de rajouter **192.168.1.2** en tant que serveur DNS afin qu'il apparaise dans le **/etc/resolv.conf** des clients.
Une fois la modification effectuée, les clients peuvent requêter le nom de domaine et sont dirigés sur la page web hebergé sur le serveur.
