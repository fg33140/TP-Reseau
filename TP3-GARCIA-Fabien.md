# I. ARP

## 1. Echange ARP

### **Générer des requêtes ARP :**  

- Pings de Marcel (10.3.1.12) vers John (10.3.1.11)
```
[root@marcel fg331]# ping 10.3.1.11 -c 4

PING 10.3.1.11 (10.3.1.11) 56(84) bytes of data.
64 bytes from 10.3.1.11: icmp_seq=1 ttl=64 time=0.469 ms
64 bytes from 10.3.1.11: icmp_seq=2 ttl=64 time=0.663 ms
64 bytes from 10.3.1.11: icmp_seq=3 ttl=64 time=0.522 ms
64 bytes from 10.3.1.11: icmp_seq=4 ttl=64 time=0.696 ms

--- 10.3.1.11 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3063ms
rtt min/avg/max/mdev = 0.469/0.587/0.696/0.094 ms
```

- Observation des tables ARP
  - ARP Marcel :
  ```
  [root@marcel fg331]# ip neigh show

  10.3.1.254 dev enp0s3 lladdr 0a:00:27:00:00:07 REACHABLE
  10.3.1.11 dev enp0s3 lladdr 08:00:27:17:b5:61 DELAY
  ```

  - ARP John :  
  ```
  [root@john fg331]# ip neigh show

  10.3.1.12 dev enp0s3 lladdr 08:00:27:40:ba:57 STALE
  10.3.1.254 dev enp0s3 lladdr 0a:00:27:00:00:07 DELAY
  ```
  On remarque que les 2 machines (John , Marcel) possèdent l'adresse MAC de l'autre + l'adresse MAC de leur passerelle qui se trouve être le PC physique.

- Adresses MAC :
  - Marcel :  
     En regardant dans la table ARP de John à la ligne :  
    ```
    10.3.1.12 dev enp0s3 lladdr 08:00:27:40:ba:57 STALE
    ```
    On déduit que l'adresse MAC de Marcel est "08:00:27:40:ba:57"  
    **PREUVE grâce à la commande "ip a" :**  
    ```
    [root@marcel fg331]# ip a

    2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
        link/ether 08:00:27:40:ba:57 brd ff:ff:ff:ff:ff:ff
        inet 10.3.1.12/24 brd 10.3.1.255 scope global noprefixroute enp0s3
        valid_lft forever preferred_lft forever
        inet6 fe80::a00:27ff:fe40:ba57/64 scope link
        valid_lft forever preferred_lft forever
    ```
    Regardons la ligne :  
    ```
    link/ether 08:00:27:40:ba:57 brd ff:ff:ff:ff:ff:ff
    ```
    On retrouve bien l'adresse MAC trouvé dans la table ARP de John "08:00:27:40:ba:57"  

  - John :  
    En regardant la table ARP de Marcel à la ligne :  
    ```
    10.3.1.11 dev enp0s3 lladdr 08:00:27:17:b5:61 DELAY  
    ```
    On déduit que l'adresse MAC de Marcel est "08:00:27:17:b5:61"  
    **PREUVE grâce à la commande "ip a" :**  
    ```
    [root@john fg331]# ip a

    2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
      link/ether 08:00:27:17:b5:61 brd ff:ff:ff:ff:ff:ff
      inet 10.3.1.11/24 brd 10.3.1.255 scope global noprefixroute enp0s3
       valid_lft forever preferred_lft forever
      inet6 fe80::a00:27ff:fe17:b561/64 scope link
       valid_lft forever preferred_lft forever
    ```
    Regardons la ligne :  
    ```
    link/ether 08:00:27:17:b5:61 brd ff:ff:ff:ff:ff:ff
    ```
    On retrouve bien l'adresse MAC trouvé dans la table ARP de John "08:00:27:17:b5:61"  

## 2. Analyse de trames

- Commande utilisé :  
 ```
 [root@john fg331]# tcpdump -i enp0s3 -c 10 -w /root/ARP-Ping-VMs.pcap not port 22
 ```  
  [fichier obtenue](/wireshark/tp2_arp.pcapng) contenant un ARP request et ARP reply (les 2 premières trames) suivis des pings

# II. Routage

## 1. Mise en place du routage

### Activer le routage sur le noeud "router" :

```
[root@router fg331]# firewall-cmd --add-masquerade --zone=public --permanent
```

### Ajouter les routes statiques nécessaire pour que "john" et "marcel" puissent se ping :  

Ajout des GATEWAY sur john et marcel de manière permanente en créant un fichier "ifcfg-enp0s3" dans "/etc/sysconfig/network-scripts/"  

- john :  
 ```
 [root@john ~]# cat /etc/sysconfig/network-scripts/ifcfg-enp0s3
DEVICE=enp0s3

BOOTPROTO=static
ONBOOT=yes

IPADDR=10.3.1.11
NETMASK=255.255.255.0

GATEWAY=10.3.1.254
```

- marcel :  
```
[root@marcel ~]# cat /etc/sysconfig/network-scripts/ifcfg-enp0s3
DEVICE=enp0s3

BOOTPROTO=static
ONBOOT=yes

IPADDR=10.3.2.12
NETMASK=255.255.255.0

GATEWAY=10.3.2.254
```

## 2. Analyse de trames  

### Analyse des échanges ARP :  

- Vider tables ARP :  
```
ip n flush all
```

- Ping de john vers marcel :  
```
[root@john fg331]# ping 10.3.2.12

PING 10.3.2.12 (10.3.2.12) 56(84) bytes of data.
64 bytes from 10.3.2.12: icmp_seq=1 ttl=63 time=0.692 ms
64 bytes from 10.3.2.12: icmp_seq=2 ttl=63 time=1.07 ms
64 bytes from 10.3.2.12: icmp_seq=3 ttl=63 time=1.03 ms
64 bytes from 10.3.2.12: icmp_seq=4 ttl=63 time=1.24 ms
```

- Observation des tables ARP :  
  - john :
   ```
   [root@john ~]# ip n s

   10.3.1.254 dev enp0s3 lladdr 08:00:27:a4:35:32 STALE
   ```
  - marcel :
   ```
   [root@marcel ~]# ip n s

   10.3.2.254 dev enp0s3 lladdr 08:00:27:bc:2f:73 STALE
   ```
  - router :
   ```
   [root@router fg331]# ip n s

   10.3.1.11 dev enp0s3 lladdr 08:00:27:17:b5:61 STALE
   10.3.2.12 dev enp0s8 lladdr 08:00:27:40:ba:57 STALE
   ```  
  
- Déduction des échanges ARP  
  Il y aura surement 2 échanges ARP :  
   - Entre John et router (pour que john conaisse la MAC de sa gateway)
   - Entre Router et marcel (pour que le router sache la MAC de marcel)  



| ordre | type trame  | IP source | MAC source              | IP destination | MAC destination            |
|-------|-------------|-----------|-------------------------|----------------|----------------------------|
| 1     | Requête ARP | x         | `john` `08:00:27:17:b5:61` | x              | Broadcast `FF:FF:FF:FF:FF` |
| 2     | Réponse ARP | x         | `router` `08:00:27:a4:35:32`                       | x              | `john` `08:00:27:17:b5:61`    |
| 3     | Requête ARP | x         | `router` `08:00:27:bc:2f:73` | x              | Broadcast `FF:FF:FF:FF:FF` |
| 4     | Réponse ARP | x         | `marcel` `08:00:27:40:ba:57`                       | x              | `router` `08:00:27:bc:2f:73`    |
| 5     | Ping        | `john` `10.3.1.11`         | `john` `08:00:27:17:b5:61`                       | `marcel` `10.3.2.12`              | `router` `08:00:27:a4:35:32`                          |
| 6     | Ping        | `router` `10.3.2.254`         | `router` `08:00:27:bc:2f:73`                       | `marcel` `10.3.2.12`              | `marcel` `08:00:27:40:ba:57`                          |
| 7     | Pong        | `marcel` `10.3.2.12`         | `marcel` `08:00:27:40:ba:57`                       | `router` `10.3.2.254`              | `router` `08:00:27:bc:2f:73`                          |
| 8     | Pong        |  `marcel` `10.3.2.12`        | `router` `08:00:27:a4:35:32`                       | `john` `10.3.1.11`              | `john` `08:00:27:17:b5:61`                          |  


- **Capture sur Marcel : [`tp2_routage_marcel.pcapng`](./pcp/tp2_routage_marcel.pcapng)**

## 3. Accès Internet

### Donnez un accès internet à vos machines :

- Ajouter une route par défault à john et marcel :  
  Il n'y a rien besoin de changer par rapport à la partie d'avant (II. 1.Mise en place du routage), les changements effectué sont permanents (car écris dans le ficher "/etc/sysconfig/network-scripts/ifcfg-enp0s3"). La route par défault de john et marcel est bien vers le router.  

  Preuve par le ping :  
   - john :  
   ```
   [root@john fg331]# ping 8.8.8.8

    PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
    64 bytes from 8.8.8.8: icmp_seq=1 ttl=113 time=17.0 ms
    64 bytes from 8.8.8.8: icmp_seq=2 ttl=113 time=16.2 ms
    64 bytes from 8.8.8.8: icmp_seq=3 ttl=113 time=17.5 ms
    64 bytes from 8.8.8.8: icmp_seq=4 ttl=113 time=15.7 ms
   ``` 
   - marcel :  
   ```
   [fg331@marcel ~]$ ping 8.8.8.8
    PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
    64 bytes from 8.8.8.8: icmp_seq=1 ttl=113 time=14.6 ms
    64 bytes from 8.8.8.8: icmp_seq=2 ttl=113 time=17.9 ms
    64 bytes from 8.8.8.8: icmp_seq=3 ttl=113 time=14.9 ms
    64 bytes from 8.8.8.8: icmp_seq=4 ttl=113 time=18.6 ms
   ```

- DNS :  
  Pour l'ajout d'un serveur DNS, on va aussi l'inscrire dans le fichier /etc/sysconfig/network-scripts/ifcfg-enp0s3 pour le rendre permanent. En ajoutant la ligne "DNS1=1.1.1.1"
   - john :  
   ```
   [root@john fg331]# cat /etc/sysconfig/network-scripts/ifcfg-enp0s3
    DEVICE=enp0s3

    BOOTPROTO=static
    ONBOOT=yes

    IPADDR=10.3.1.11
    NETMASK=255.255.255.0

    GATEWAY=10.3.1.254
    DNS1=1.1.1.1
    ```
     Preuve :  
     ```
     [root@john fg331]# dig google.com

    ; <<>> DiG 9.16.23-RH <<>> google.com
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 1532
    ;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

    ;; OPT PSEUDOSECTION:
    ; EDNS: version: 0, flags:; udp: 1232
    ;; QUESTION SECTION:
    ;google.com.                    IN      A

    ;; ANSWER SECTION:
    google.com.             108     IN      A       142.250.179.110

    ;; Query time: 22 msec
    ;; SERVER: 1.1.1.1#53(1.1.1.1)
    ;; WHEN: Mon Oct 10 19:07:19 CEST 2022
    ;; MSG SIZE  rcvd: 55
     ```
     ```
     [root@john fg331]# ping ynov.com

    PING ynov.com (172.67.74.226) 56(84) bytes of data.
    64 bytes from 172.67.74.226 (172.67.74.226): icmp_seq=1 ttl=54 time=15.5 ms
    64 bytes from 172.67.74.226 (172.67.74.226): icmp_seq=2 ttl=54 time=17.1 ms
    64 bytes from 172.67.74.226 (172.67.74.226): icmp_seq=3 ttl=54 time=17.0 ms
    64 bytes from 172.67.74.226 (172.67.74.226): icmp_seq=4 ttl=54 time=17.7 ms
     ```
    - Marcel :  
    ```
    [fg331@marcel ~]$ cat /etc/sysconfig/network-scripts/ifcfg-enp0s3

    DEVICE=enp0s3

    BOOTPROTO=static
    ONBOOT=yes

    IPADDR=10.3.2.12
    NETMASK=255.255.255.0

    GATEWAY=10.3.2.254
    DNS1=1.1.1.1
    ```
    Preuve :  
    ```
    [fg331@marcel ~]$ dig google.com

    ; <<>> DiG 9.16.23-RH <<>> google.com
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 15171
    ;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

    ;; OPT PSEUDOSECTION:
    ; EDNS: version: 0, flags:; udp: 1232
    ;; QUESTION SECTION:
    ;google.com.                    IN      A

    ;; ANSWER SECTION:
    google.com.             266     IN      A       142.250.75.238

    ;; Query time: 31 msec
    ;; SERVER: 1.1.1.1#53(1.1.1.1)
    ;; WHEN: Mon Oct 10 19:11:15 CEST 2022
    ;; MSG SIZE  rcvd: 55
    ```
    ```
    [fg331@marcel ~]$ ping ynov.com

    PING ynov.com (172.67.74.226) 56(84) bytes of data.
    64 bytes from 172.67.74.226 (172.67.74.226): icmp_seq=1 ttl=54 time=18.2 ms
    64 bytes from 172.67.74.226 (172.67.74.226): icmp_seq=2 ttl=54 time=18.6 ms
    64 bytes from 172.67.74.226 (172.67.74.226): icmp_seq=3 ttl=54 time=16.0 ms
    64 bytes from 172.67.74.226 (172.67.74.226): icmp_seq=4 ttl=54 time=18.3 ms
    ```

### Analyse de trames :  

| ordre | type trame | IP source          | MAC source              | IP destination | MAC destination |     |
|-------|------------|--------------------|-------------------------|----------------|-----------------|-----|
| 1     | ping       | `john` `10.3.1.12` | `john` `08:00:27:17:b5:61` | `8.8.8.8`      | `router` `08:00:27:a4:35:32`               |     |
| 2     | pong       | `8.8.8.8`                | `router` `08:00:27:a4:35:32`                     | `john` `10.3.1.12`            | `john` `08:00:27:17:b5:61`             |     |  

**Capture du ping :** [tp2_routage_internet.pcapng](/wireshark/tp2_routage_internet.pcapng)


# III. DHCP

## 1. Mise en place du serveur DHCP

- Configuration du serveur DHCP sur john :  
  - Installation packages :  
  ```
  dnf install dhcp-server
  ```
  - Editer le fichier /etc/dhcp/dhcpd.conf comme tel :  
  ```
  [root@john ~]# cat /etc/dhcp/dhcpd.conf

  default-lease-time 900;
  max-lease-time 10800;
  ddns-update-style none;
  authoritative;
  subnet 10.3.1.0 netmask 255.255.255.0 {
    range 10.3.1.12 10.3.1.253;
    option routers 10.3.1.254;
    option subnet-mask 255.255.255.0;
    option domain-name-servers 1.1.1.1;

  }
    ```  
    Avec cette configuration le service DHCP fournira :  
        - une IP en respectant l'intervale 10.3.1.12-253 : ```range 10.3.1.12 10.3.1.253; ```  
        - une gateway : ```option routers 10.3.1.254; ```  
        - l'IP d'un serveur DNS : ```option domain-name-servers 1.1.1.1; ```  
  - Ouverture port 67 :  
  ```
  firewall-cmd --permanent --add-port=67/udp
  ```
  - Activation DHCP :  
  ```
  systemctl start dhcpd
  ```

- Configurer Bob pour qu'il obtienne une IP grâce au service DHCP de John :  
  - Modifier le fichier "/etc/sysconfig/network-scrips/ifcfg-enp0s3" comme tel :  
  ```
  [root@bob fg331]# cat /etc/sysconfig/network-scripts/ifcfg-enp0s3
  DEVICE=enp0s3

  BOOTPROTO=dhcp
  ONBOOT=yes
  ```

Une fois les 2 machines configuré on voit que Bob à maintenant une IP :  
```
[root@bob fg331]# ip a

2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:a4:35:31 brd ff:ff:ff:ff:ff:ff
    inet 10.3.1.12/24 brd 10.3.1.255 scope global dynamic noprefixroute enp0s3
       valid_lft 782sec preferred_lft 782sec
    inet6 fe80::a00:27ff:fea4:3531/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```
  
Grâce aux lignes dans le fichier "/etc/dhcp/dhcpd.conf":  
```
option routers 10.3.1.254;
option domain-name-servers 1.1.1.1;
```
On fournis une route par défault (IP du routeur) et un server DNS à utiliser (1.1.1.1)

**Preuves :** 

Récupération nouvelle IP :  
```
[root@bob fg331]# dhclient
```

- IP :  
 ```
 [root@bob fg331]# ip a

2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:a4:35:31 brd ff:ff:ff:ff:ff:ff
    inet 10.3.1.12/24 brd 10.3.1.255 scope global dynamic noprefixroute enp0s3
      valid_lft 736sec preferred_lft 736sec
    inet6 fe80::a00:27ff:fea4:3531/64 scope link
      valid_lft forever preferred_lft forever
 ```
 ```
 [root@bob fg331]# ping 10.3.1.254

  PING 10.3.1.254 (10.3.1.254) 56(84) bytes of data.
  64 bytes from 10.3.1.254: icmp_seq=1 ttl=64 time=0.435 ms
  64 bytes from 10.3.1.254: icmp_seq=2 ttl=64 time=0.643 ms
  64 bytes from 10.3.1.254: icmp_seq=3 ttl=64 time=0.635 ms
  ^C
  --- 10.3.1.254 ping statistics ---
  3 packets transmitted, 3 received, 0% packet loss, time 2045ms
  rtt min/avg/max/mdev = 0.435/0.571/0.643/0.096 ms
 ```

- Route par défault :  
```
[root@bob fg331]# ip route show

default via 10.3.1.254 dev enp0s3 proto dhcp src 10.3.1.12 metric 100
10.3.1.0/24 dev enp0s3 proto kernel scope link src 10.3.1.12 metric 100
```
```
[root@bob fg331]# ping 8.8.8.8

PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=113 time=15.8 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=113 time=13.4 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=113 time=16.6 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=113 time=18.0 ms
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 13.418/15.977/18.029/1.672 ms
```

- DNS :  
```
[root@bob fg331]# dig ynov.com

; <<>> DiG 9.16.23-RH <<>> ynov.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 13478
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;ynov.com.                      IN      A

;; ANSWER SECTION:
ynov.com.               99      IN      A       104.26.11.233
ynov.com.               99      IN      A       172.67.74.226
ynov.com.               99      IN      A       104.26.10.233

;; Query time: 22 msec
;; SERVER: 1.1.1.1#53(1.1.1.1)
;; WHEN: Tue Oct 11 14:56:50 CEST 2022
;; MSG SIZE  rcvd: 85
```
```
[root@bob fg331]# ping ynov.com

PING ynov.com (104.26.10.233) 56(84) bytes of data.
64 bytes from 104.26.10.233 (104.26.10.233): icmp_seq=1 ttl=54 time=13.8 ms
64 bytes from 104.26.10.233 (104.26.10.233): icmp_seq=2 ttl=54 time=19.2 ms
64 bytes from 104.26.10.233 (104.26.10.233): icmp_seq=3 ttl=54 time=16.3 ms
64 bytes from 104.26.10.233 (104.26.10.233): icmp_seq=4 ttl=54 time=15.6 ms
^C
--- ynov.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3006ms
rtt min/avg/max/mdev = 13.845/16.237/19.166/1.916 ms
```

## 2. Analyse de trames :  

[tp2_dhcp.pcapng](/wireshark/tp2_dhcp.pcapng)