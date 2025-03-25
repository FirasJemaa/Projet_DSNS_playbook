## **Tableau d’adressage**

| **Réseau / VLAN**                                               | **Adresse réseau** | **CIDR** | **Masque**      | **Broadcast** | **Passerelle** | **Nb total d’IP** | **Nb IP utiles** | **Plage DHCP**               | **IPs réservées (statiques)**                                                           | **Observations**                                     |
| --------------------------------------------------------------- | ------------------ | -------- | --------------- | ------------- | -------------- | ----------------- | ---------------- | ---------------------------- | --------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| <span style="color:green">**IT  NETWORK **<br>(VLAN 10)</span>  | 172.16.10.0        | /27      | 255.255.255.224 | 172.16.10.31  | 172.16.10.1    | 32                | 30               | 172.16.10.2 → 172.16.10.30   | - IT-ANSIBLE :  172.16.10.10- IT-GRAFANA : 172.16.10.1                                  | Réseau d’administration et supervision               |
| <span style="color:green">**DMZ NETWORK**<br>(Réseaux)</span>   | 10.10.10.0         | /28      | 255.255.255.240 | 10.10.10.15   | 10.10.10.1     | 16                | 14               | Statique                     | DMZ-SMTP : 10.10.10.3 DMZ-DNS : 10.10.10.4 DMZ-RPROXY : 10.10.10.5 DMZ-WEB : 10.10.10.2 | La DMZ doit être sécurisée avec ACL/pare-feu         |
| <span style="color:green">**SRV NETWORK** <br>(VLAN 50)</span>  | 172.16.50.0        | /29      | 255.255.255.248 | 172.16.50.7   | 172.16.50.1    | 8                 | 6                | 172.16.50.2 → 172.16.50.6    | - SRV-MAIL : 172.16.50.4- SRV-INTRANET : 172.16.50.3- SRV-AD01 : 172.16.50.2            | Serveurs internes, accès restreint                   |
| <span style="color:green">**CORE NETWORK** <br>(VLAN 60)</span> | 172.16.60.0        | /29      | 255.255.255.252 | 172.16.60.3   | 172.16.60.1    | 8                 | 6                | 172.16.60.2-5                | - PC : 172.16.60.2 <br>- BORDER-RT : 172.16.60.1                                        | Réseau dédié au routage entre équipements principaux |
| <span style="color:green">STORM-NET</span>                      | 10.11.11.0         | /30      | 255.255.255.252 | 10.11.11.3    | 10.11.11.1     | 4                 | 2                |                              | - Border-RT (DMZ1) : 10.11.11.1 - DMZ-RT (OUT) : 10.11.11.2                             | Réseau entre les deux pare-feu                       |
| <span style="color:green">**Production** <br>(VLAN 20)</span>   | 192.168.20.0       | /28      | 255.255.255.240 | 192.168.20.15 | 192.168.20.1   | 16                | 14               | 192.168.20.2 - 192.168.20.14 |                                                                                         |                                                      |
| <span style="color:green">**Commerciaux** <br>(VLAN 30)</span>  | 192.168.30.0       | /28      | 255.255.255.240 | 192.168.30.15 | 192.168.30.1   | 16                | 14               | 192.168.30.2 - 192.168.30.14 |                                                                                         |                                                      |
| <span style="color:green">**Production** <br>(VLAN 40)</span>   | 192.168.40.0       | /26      | 255.255.255.192 | 192.168.40.63 | 192.168.40.1   | 32                | 30               | 192.168.40.2 - 192.168.40.62 | 192.168.40.63                                                                           |                                                      |

---
## **Compléments et Explications**
1. **NET-IT**
    - Ce réseau est utilisé pour l’administration et le monitoring (Ansible, Grafana).
    - DHCP limité car peu de machines, majoritairement des serveurs.
2. **NET-DMZ**
    - Destiné aux services accessibles depuis Internet (SMTP, DNS, Web, Proxy).
    - Pas de DHCP pour des raisons de sécurité (tout est en IP statique).
    - Sécurité renforcée (ACL, Firewall, IDS/IPS si nécessaire).
3. **NET-LAN**
    - Réseau pour les utilisateurs (Direction, Administratif, Commerciaux, Production).
    - Utilisation de DHCP pour attribuer les IP dynamiquement aux postes utilisateurs.
    - Peut être subdivisé en plusieurs VLANs pour isoler les départements.
4. **NET-SRV**
    - Contient les serveurs internes (AD, Mail, Intranet).
    - Nécessite des règles strictes pour éviter que le LAN accède directement aux serveurs sans contrôle.
5. **CORE-NET**
    - Réseau d’interconnexion entre les routeurs et switches.
    - Utilisation de **/30** pour minimiser le gaspillage d’IP.
    - Pas de DHCP ici, uniquement des adresses fixes.

---
## **Suggestions d’améliorations**

1. **Ajout d’un VLAN Management (ex. VLAN 99)**
    - Pour isoler la gestion des switches et routeurs (accès SSH, SNMP).
    - Ex. `192.168.99.0/28`.
2. **Redondance des liens CORE-NET**
    - Utiliser des protocoles de haute disponibilité comme **HSRP, VRRP ou GLBP** sur les routeurs principaux.
3. **Firewall pour NET-DMZ**
    - Pour mieux filtrer les accès vers la DMZ depuis Internet et le LAN.
    - Ex. un pare-feu comme pfSense ou un Cisco ASA.
4. **Ajout de VLAN spécifiques pour les services**
    - VLAN VoIP pour téléphonie.
    - VLAN IoT si présence d’équipements industriels.

---

## **Conclusion**

- Ce **tableau d’adressage** est **directement basé sur l’image**, avec une segmentation claire des réseaux.
- Il inclut **les VLANs, les IPs, le DHCP, les passerelles et les équipements associés**.
- Il reste **évolutif** (possibilité d’ajouter des VLANs, des adresses IP en fonction des besoins).
- La sécurité est prise en compte avec une **séparation des rôles** entre DMZ, IT, LAN et Serveurs.

---

---

## VLANS

| VLAN ID | Nom du VLAN                                                   | Adresse Réseau | CIDR | Masque          | Broadcast      | Passerelle (J'ai mis vlan ??) | Nb Total IPs | Nb IPs Utiles | Plage DHCP                      | Notes                                           |
| ------- | ------------------------------------------------------------- | -------------- | ---- | --------------- | -------------- | ----------------------------- | ------------ | ------------- | ------------------------------- | ----------------------------------------------- |
| 10      | NET-IT                                                        | 172.16.86.0    | 27   | 255.255.255.224 | 172.16.86.31   | 172.16.86.1                   | 32           | 30            | 172.16.86.10 - 172.16.86.20     | Admin & Monitoring (Ansible, Grafana)           |
| 20      | <span style="color:orange">NET-DMZ<span style="color:orange"> | 192.168.85.0   | 28   | 255.255.255.240 | 172.16.84.15   | 172.16.84.1                   | 16           | 14            | 192.168.85.(10 - 20)            | Zone DMZ (SMTP, DNS, Web, Reverse Proxy)        |
| 31      | Direction & Administratif                                     | 192.168.90.0   | 26   | 255.255.255.192 | 192.168.90.63  | 192.168.90.1                  | 64           | 62            | 192.168.90.10 - 192.168.90.25   | Utilisateurs et administration générale         |
| 32      | Commercial                                                    | 192.168.90.64  | 26   | 255.255.255.192 | 192.168.90.127 | 192.168.90.65                 | 64           | 62            | 192.168.90.74 - 192.168.90.88   | Réseau dédié aux commerciaux                    |
| 33      | Production                                                    | 192.168.90.128 | 26   | 255.255.255.192 | 192.168.90.191 | 192.168.90.129                | 64           | 62            | 192.168.90.138 - 192.168.90.178 | Réseau de production et équipements industriels |
| 40      | NET-SRV                                                       | 172.16.85.0    | 27   | 255.255.255.224 | 172.16.85.31   | 172.16.85.1                   | 32           | 30            | 172.16.85.13 - 172.16.85.20     | Serveurs Internes (AD, Mail, Intranet)          |
| 50      | CORE-NET                                                      | 10.10.10.0     | 30   | 255.255.255.252 | 10.10.10.3     | 10.10.10.1                    | 4            | 2             | N/A (Fixe)                      | Interconnexion Routeurs/Switches                |
| 99      | MANAGEMENT                                                    | 192.168.99.0   | 28   | 255.255.255.240 | 192.168.99.15  | 192.168.99.1                  | 16           | 14            | 192.168.99.10 - 192.168.99.14   | Gestion des équipements réseau (SSH, SNMP)      |
|         | ENTRE LES DEUX STORMSHIELD                                    |                |      |                 |                |                               |              |               |                                 |                                                 |


VOIR COMMENT MARCHE UN RESEAU ET UN SOUS RESEAU : exemple avec le NET-LAN et les 3 sous réseaux

### ==Message SAMY :==

Pour : 
— GRP-SECU-COM
— GRP-SECU-DIR
— GRP-SECU-PROD

C'est des utilisateurs standards. 

Pour GRP-SECU-IT c'est l'équipe IT donc forcément des accès étendus après on peut très bien faire un GRP-SECU-PRINCIPAL pour appliquer des stratégies qui concernent tout le monde puis faire des stratégies spécifiques pour chacun des groupes.

Enfin pour les admins (IT) la bonne pratique veut que chaque membre d'une équipe IT dispose d'un compte normal nominatif et d'un compte admin nominatif (on sépare les deux), la personne est en suite censée utilisée son compte admin que quand elle doit faire des opérations d'administration. Généralement on nomme ces comptes adm-prenom.nom

Petite précision pour la documentation : 
Je prends l'exemple de l'AD, vous allez installer l'OS, le mettre en IP fixe, lui donner un nom, installer le rôle AD etc... Tout ça vous n'avez pas besoin de le documenter en mode tuto, c'est inutile, il suffit juste de documenter les paramétrages effectués (sous forme de tableau par exemple) avec les infos pertinentes : 
Nom d'hôte
Adresse IP
Nom de la fôret 
Niveau fonctionnel 
Mot de passe de restauration des services AD
etc... 

Ensuite pour le paramétrage de l'AD (création des OU, groupes, gpo etc...) idem pas besoin de tuto pas à pas, dans la mesure du possible vous allez faire ça avec Ansible, je préfère que vous commentiez vos playbook pour comprendre ce qui se passe à chaque étape du playbook c'est plus pertinant puis expliquer comment lancer le playbook.
Si c'est des actions manuelles vous pouvez synthétiser en indiquant où se fait le paramétrage et ce que vous avez paramétré et pourquoi, mais pas besoin de faire de tuto pas à pas (j'ai cliqué là, on clique ici etc...) ça c'est pas utile.