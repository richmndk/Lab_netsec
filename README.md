# Lab_netsec

![Cisco Packet Tracer](https://img.shields.io/badge/Cisco-Packet%20Tracer-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![Réseau](https://img.shields.io/badge/Réseau-Configuration-blue?style=for-the-badge)
![Sécurité](https://img.shields.io/badge/Sécurité-Cybersécurité-red?style=for-the-badge)

# TP Réseau & Sécurité — Interconnexion et Sécurisation avec Cisco Packet Tracer

##  But du TP

Ce TP a pour objectif de mettre en place une petite architecture réseau composée de deux LAN reliés par un routeur, avec deux switches et quatre PC. Le but est double :

1. **Faire communiquer** tous les équipements entre eux (connectivité de base).
2. **Sécuriser** l'ensemble : accès aux équipements, ports inutilisés, et filtrage du trafic entre certains PC grâce à une ACL.

---

## 🖧 Topologie du réseau

| Équipement | Interface | Adresse IP | Masque | Connecté à |
|---|---|---|---|---|
| Router0 | G0/0 | 192.168.1.1 | /24 | Switch0 |
| Router0 | G0/1 | 192.168.2.1 | /24 | Switch1 |
| Switch0 | Fa0/1 | — | — | PC0 |
| Switch0 | Fa0/2 | — | — | PC1 |
| Switch0 | Fa0/20 | — | — | Switch1 |
| Switch1 | Fa0/1 | — | — | PC2 |
| Switch1 | Fa0/2 | — | — | PC3 |
| PC0 | Fa0 | 192.168.1.10 | /24 | gw 192.168.1.1 |
| PC1 | Fa0 | 192.168.1.11 | /24 | gw 192.168.1.1 |
| PC2 | Fa0 | 192.168.2.10 | /24 | gw 192.168.2.1 |
| PC3 | Fa0 | 192.168.2.11 | /24 | gw 192.168.2.1 |

---

## 1️⃣ Configuration — Router0

```
enable
configure terminal
hostname Router0

interface g0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
 exit

interface g0/1
 ip address 192.168.2.1 255.255.255.0
 no shutdown
 exit
```

**Explication simple** : le routeur a une patte dans chaque réseau (G0/0 et G0/2). Comme les deux réseaux sont directement branchés dessus, il n'y a pas besoin de routage statique : le routeur sait déjà comment aller de l'un à l'autre.

---

## 2️⃣ Configuration — Switch0

```
enable
configure terminal
hostname Switch0

interface range fa0/1-2
 switchport mode access
 exit
```

**Explication simple** : Fa0/1 et Fa0/2 sont les ports branchés aux PC (PC0 et PC1). On les met en mode "access" car chaque port ne sert qu'à un seul appareil, pas de VLAN multiple ici.

---

## 3️⃣ Configuration — Switch1

```
enable
configure terminal
hostname Switch1

interface range fa0/1-2
 switchport mode access
 exit
```

**Explication simple** : même logique que Switch0, mais pour PC2 et PC3.

---

## 4️⃣ Sécurisation

### a) Mots de passe et accès (sur Router0, Switch0 et Switch1)

```
enable secret Cisco123!
username admin secret Admin123!
service password-encryption

line console 0
 password Cisco123!
 login
 exit

line vty 0 4
 login local
 transport input ssh
 exit
```

**Explication simple** : on protège l'accès à chaque équipement avec un mot de passe, et on chiffre les mots de passe stockés pour qu'ils ne soient pas visibles en clair.

### b) SSH (sur Router0, Switch0 et Switch1)

```
ip domain-name tp-securite.local
crypto key generate rsa
ip ssh version 2
```

**Explication simple** : SSH permet de se connecter à distance de façon chiffrée, contrairement à Telnet qui envoie tout en clair (mot de passe compris).

### c) Bannière de connexion (sur Router0, Switch0 et Switch1)

```
banner motd #Accès réservé - Toute tentative non autorisée sera loggée#
```

### d) Port Security (sur Switch0 et Switch1 uniquement)

```
interface range fa0/1-2
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 exit
```

**Explication simple** : ça empêche qu'un pirate débranche un PC autorisé et branche son propre appareil sur le même port. Si une adresse MAC inconnue apparaît, le port se coupe automatiquement.

### e) Désactivation des ports inutilisés (sur Switch0 et Switch1)

```
interface range fa0/3-19
 shutdown
 exit
interface range fa0/21-24
 shutdown
 exit
```

**Explication simple** : on coupe tous les ports non utilisés pour éviter qu'un intrus branche un appareil dessus. ⚠️ Attention à ne pas couper Fa0/20, il sert au lien entre Switch0 et Switch1.

### f) ACL — Restriction du trafic (sur Router0)

**Objectif** : PC2 (192.168.2.10) ne doit pouvoir parler qu'à PC1 (192.168.1.11), et à personne d'autre.

```
configure terminal
access-list 110 permit ip host 192.168.2.10 host 192.168.1.11
access-list 110 deny ip host 192.168.2.10 any
access-list 110 permit ip any any

interface g0/1
 ip access-group 110 in
 exit
```

**Explication simple** :
- La 1ère ligne autorise PC2 à parler uniquement à PC1.
- La 2ème ligne bloque tout le reste venant de PC2.
- La 3ème ligne laisse passer tout le trafic normal des autres PC (non concernés par la restriction).
- L'ordre des lignes est très important : elles sont lues de haut en bas.

---

## 5️⃣ Sauvegarde de la configuration

Sur **Router0, Switch0 et Switch1** :

```
copy run start
```

---

## 6️⃣ Vérifications

| Test | Depuis | Résultat attendu |
|---|---|---|
| `ping 192.168.1.11` | PC0 |  réussit |
| `ping 192.168.2.10` | PC0 |  réussit |
| `ping 192.168.1.11` | PC2 |  réussit (règle spéciale ACL) |
| `ping 192.168.1.10` | PC2 |  échoue (bloqué par ACL) |
| `ping 192.168.1.10` | PC3 |  réussit (non concerné par l'ACL) |
| `show ip interface brief` | Router0 | Toutes les interfaces en up/up |
| `show port-security` | Switch0 / Switch1 | Ports en secure-up |
| `show access-lists` | Router0 | Affiche l'ACL 110 |

---

##  Conclusion

Ce TP permet de comprendre les bases essentielles d'une infrastructure réseau sécurisée :
- La connectivité inter-réseaux via un routeur.
- La protection des accès aux équipements (mots de passe, SSH).
- La sécurisation physique des ports (Port Security).
- Le filtrage précis du trafic entre machines grâce aux ACL.

---

**Auteur** : Richmond, Network & security Technician 
