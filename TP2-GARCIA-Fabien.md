Ce Tp à été réalisé avec Raphael MENARD

# I. Setup IP

L'adresse réseau est 10.128.128.0/26 et l'adresse broadcast est 10.128.128.63  

```bash
netsh interface ipv4 set address name="Ethernet" static 10.128.128.1 255.255.255.192
```
La machine A aura l'ip 10.128.128.1/26 (configuré via la commande ci-dessus) et la machine B aura l'ip 10.128.128.2/26  

```bash
ping 10.128.128.1

Envoi d’une requête 'Ping'  10.128.128.1 avec 32 octets de données :
Réponse de 10.128.128.1 : octets=32 temps<1ms TTL=128
Réponse de 10.128.128.1 : octets=32 temps=1 ms TTL=128
Réponse de 10.128.128.1 : octets=32 temps=1 ms TTL=128
Réponse de 10.128.128.1 : octets=32 temps=2 ms TTL=128

Statistiques Ping pour 10.128.128.1:
    Paquets : envoyés = 4, reçus = 4, perdus = 0 (perte 0%),
Durée approximative des boucles en millisecondes :
    Minimum = 0ms, Maximum = 2ms, Moyenne = 1ms
```
Le changement d'ip fonctionne

Il y a 2 types de paquets ICMP utilisés pour nos pings, le type 0 pour un reply et le type 8 pour une request [cf.Ping.pcapng](wireshark/Ping.pcapng)

# II. ARP my bro

## Affichage table ARP
```bash
arp -a

Interface : 10.128.128.2 --- 0x3b
  Adresse Internet      Adresse physique      Type
  10.128.128.1          00-e0-4c-68-00-67     dynamique
  10.128.128.63         ff-ff-ff-ff-ff-ff     statique
  224.0.0.22            01-00-5e-00-00-16     statique
  224.0.0.251           01-00-5e-00-00-fb     statique
  224.0.0.252           01-00-5e-00-00-fc     statique
  239.255.255.250       01-00-5e-7f-ff-fa     statique
```

## Adresse MAC de notre binome

```bash
Adresse Internet      Adresse physique      Type
  10.128.128.1          00-e0-4c-68-00-67     dynamique
```
L'adresse MAC de notre binome est donc 00:e0:4c:68:00:67

## Adresse MAC de la gateway

```bash
arp -a

Interface : 10.33.19.87 --- 0xc
  Adresse Internet      Adresse physique      Type
  10.33.19.192          f4-7b-09-a6-81-0a     dynamique
  10.33.19.254          00-c0-e7-e0-04-4e     dynamique
  224.0.0.22            01-00-5e-00-00-16     statique
  224.0.0.251           01-00-5e-00-00-fb     statique
  224.0.0.252           01-00-5e-00-00-fc     statique
  239.255.255.250       01-00-5e-7f-ff-fa     statique
  255.255.255.255       ff-ff-ff-ff-ff-ff     statique
```
En sachant que l'IP de la passerelle du réseau YNOV est 10.33.19.254  
Il nous suffit de regarder la ligne :  

```bash
Adresse Internet      Adresse physique      Type
  10.33.19.254          00-c0-e7-e0-04-4e     dynamique
```
L'adresse MAC de la passerelle est 00:c0:e7:e0:04:4e

## Manipuler la table ARP
### Vider la table ARP :  
On utilise la commande suivante pour pouvoir vider nos tables arp 
```bash
netsh interface ip delete arpcache
```

```bash
arp -a

Interface : 10.33.19.87 --- 0xc
  Adresse Internet      Adresse physique      Type
  224.0.0.22            01-00-5e-00-00-16     statique

Interface : 10.128.128.2 --- 0x3b
  Adresse Internet      Adresse physique      Type
  224.0.0.22            01-00-5e-00-00-16     statique
```
Ce qui permet effectivement de vider les tables ARP de nos interfaces réseau Ethernet et Wifi (Ynov)

### Remplissage de nos tables ARP :  


Après un nouveau ping (vers le binome, donc dans notre petit LAN) on obtient ça :
```bash
arp -a

Interface : 10.33.19.87 --- 0xc
  Adresse Internet      Adresse physique      Type
  10.33.19.192          f4-7b-09-a6-81-0a     dynamique
  10.33.19.254          00-c0-e7-e0-04-4e     dynamique
  224.0.0.22            01-00-5e-00-00-16     statique
  239.255.255.250       01-00-5e-7f-ff-fa     statique

Interface : 10.128.128.2 --- 0x3b
  Adresse Internet      Adresse physique      Type
  10.128.128.1          00-e0-4c-68-00-67     dynamique
  224.0.0.22            01-00-5e-00-00-16     statique
```

```bash
Interface : 10.128.128.2 --- 0x3b
  Adresse Internet      Adresse physique      Type
  10.128.128.1          00-e0-4c-68-00-67     dynamique
  224.0.0.22            01-00-5e-00-00-16     statique
```
On peut voir ici que l'adresse MAC de notre binome a bien été rajouté.  
*Remarque* : la table ARP du réseau YNOV s'est aussi remplie 

### Wizeshark ARP :
[wireshark de l'ARP](wireshark/ARP-Ping.pcapng)  
On vide la table ARP pour forcer une requete ARP quand on va ping 10.128.128.2, qui seront les 2 premières trames du fichier ci-dessus :  
 - Pour la premiere trame (adresse source : 00:e0:4c:68:00:67, adresse destination : Broadcast ,ff:ff:ff:ff:ff:ff), l'adresse MAC source est celle de la machine 10.128.128.1 qui fait la demande à destination de "broadcast" (ça signifie que la trame est destiné à tout le monde)  
 - Pour la deuxieme trame (adresse source : 00:e0:4c:68:01:54, adresse destination : 00:e0:4c:68:00:67), l'adresse MAC source est celle de la machine 10.128.128.2 à destination de la machine 10.128.128.1

On voit ensuite les trames ICMP (le ping) puis, chose intéressante, la machine 10.128.128.2 effectue elle aussi une requête ARP, surement suite à la découverte de l'IP d'une nouvelle machine (10.128.128.1) dans le réseau qui n'est pas encore associé à une adresse MAC.

# III. DHCP you too my brooo  

Etant sur Windows, on a changé notre IP dans le réseau YNOV sans ajouté de passerelle, une fois les changements effectif on se deconnecte du réseau YNOV.  

 On re configure notre carte réseau Wi-fi pour utiliser DHCP lors d'une connection et enfin on se reconnecte sur le réseau YNOV pour capturer l'échange DHCP (DORA)

## Changement adresse IP de la carte réseau Wi-fi
```bash
netsh interface ipv4 set address name="Wi-Fi" static 10.33.19.200 255.255.252.0
```
```bash
Carte réseau sans fil Wi-Fi :

   Suffixe DNS propre à la connexion. . . :
   Adresse IPv6 de liaison locale. . . . .: fe80::854b:9205:61cc:149a%12
   Adresse IPv4. . . . . . . . . . . . . .: 10.33.19.200
   Masque de sous-réseau. . . . . . . . . : 255.255.252.0
   Passerelle par défaut. . . . . . . . . :
```
## Re configuration de la carte réseau Wi-fi
**Utilisation de DHCP lors d'une connexion** :
```bash
netsh interface ipv4 set address name="Wi-Fi" source=dhcp
```
## Résultat de la manipulation
```bash
Carte réseau sans fil Wi-Fi :

   Suffixe DNS propre à la connexion. . . :
   Adresse IPv6 de liaison locale. . . . .: fe80::854b:9205:61cc:149a%12
   Adresse IPv4. . . . . . . . . . . . . .: 10.33.19.104
   Masque de sous-réseau. . . . . . . . . : 255.255.252.0
   Passerelle par défaut. . . . . . . . . : 10.33.19.254
```
 On remarque que l'utilisation du serveur DHCP a été un succés. Maintenant voyons voir les trames :  

On peut voir [ici](wireshark/DHCP.pcapng) les différentes étapes de l'échange DHCP.
Les informations recherchées se trouvent dans la trame 2 (Offer):  
- Ip à utiliser :  
Elle se trouve dans la partie "Dynamic Host Configuration Protocol(offer)"  
A la ligne "Your (client) IP address : 10.33.19.104"  

- IP de la passerelle du réseau : 
Elle se trouve aussi dans la partie "Dynamic Host Configuration Protocol(offer)"  
Puis dans la sous-partie "Option : (3) Router"  
A la ligne "Router : 10.33.19.254"  

- Ip d'un serveur DNS joignable depuis ce réseau :  
 Elle se trouve aussi dans la partie "Dynamic Host Configuration Protocol(offer)"  
 Puis dans la sous-partie "Option : (6) Domain Name Server"  
 3 Ip différentes sont données :  

    - "Domain Name Server: 8.8.8.8"  
    - "Domain Name Server: 8.8.4.4"
    - "Domain Name Server: 1.1.1.1"

# IV. Avant-goût TCP et UDP
Comme on peut voir [ici](wireshark/youtube.pcapng) qui est une partie des nombreuses trames UDP reçu.  
L'ip sur laquelle on se connecte est : 77.136.192.84 et le port auquel on se connecte est le port 443
