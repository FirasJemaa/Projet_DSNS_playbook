# Configuration de l’IP statique sur les machines

## 1. Désactivation du DHCP sur le Stormshield (DMZ-RT)
Avant d'attribuer des adresses IP statiques, il est essentiel de désactiver le serveur DHCP pour éviter les conflits.
### Étapes :
1. **Se connecter à l’interface Web** du Stormshield via son adresse IP d’administration (`https://<ip-du-stormshield>`).
2. **Accéder aux paramètres DHCP** :
    - Aller dans **Configuration** → **Réseau** → **DHCP**.
3. **Désactiver le serveur DHCP** en décochant l’option **"Activer le service DHCP"** sur l’interface DMZ.
4. **Appliquer les changements**.
## 2. Configuration d'une adresse IP statique sur chaque machine
Chaque machine de la DMZ doit être configurée avec une adresse IP statique pour assurer une connectivité stable.
### Étapes générales :
1. **Connecter la machine au switch dédié** dans GNS3.
2. **Configurer la machine manuellement :**
    - **Clic droit sur la machine** dans GNS3 → **Configure**.
    - Aller dans l’onglet **General settings**.
    - Descendre jusqu’à **Network configuration** et cliquer sur **Edit**.
### 3. Modification du fichier de configuration réseau
**Ouvrir le fichier de configuration réseau** et activer l’IP statique, comme dans l'exemple ci-dessous :
```sh
auto eth0
iface eth0 inet static
	address 10.10.10.2         # Adresse IP de la machine
	netmask 255.255.255.240    # Masque de sous-réseau
	gateway 10.10.10.1         # Passerelle dédié
	up echo "nameserver 8.8.8.8" > /etc/resolv.conf 
	up echo "nameserver 1.1.1.1" >> /etc/resolv.conf
```

**Remarque :** Si un serveur DNS interne est utilisé, remplacer le dns préféré par `172.16.50.2` (IP du serveur DNS local) et `1.1.1.1` .
### 4. Appliquer les modifications et tester la connexion

1. **Redémarrer l’interface réseau** avec :
    ```bash
    ip link set eth0 down
    ip link set eth0 up
    ```
    
2. Vérifier l’assignation IP :
    ```bash
    ip a
    ```
    
3. **Tester la connectivité** :
    ```bash
    ping 10.10.10.1    # Tester la communication avec la passerelle
    ping 8.8.8.8         # Tester la connectivité Internet
    ```

### 5. Répéter l’opération pour les autres machines
- Modifier l’**adresse IP** pour chaque machine en respectant la plage définie.
- Assigner une **IP unique** pour éviter les conflits.
- Vérifier que chaque machine peut **communiquer avec la passerelle et les autres équipements**.

---
# Rendre la configuration persistante
1. Faire un clic droit sur la machine, ensuite cliquer sur "**Configure**"
2. Dans l'onglet "**Advanced**" nous avons le second champ qui nous renseigne : "
Additional directories to make persistent that are not included in the image VOLUMES config. One directory per line._"
3. Ajouter les répertoires suivants :
```
/bin
/boot
/etc
/gns3
/home
/lib
/lib64
/media
/mnt
/opt
/root
/run
/sbin
/srv
/tmp
/usr
/var
```

---
# Installation OpenSSH Server
### 1. Ouvrir la console du conteneur dans GNS3
1. Lancer le conteneur Ubuntu dans GNS3 (docker Ubuntu guest).
2. Faire un clic droit > **Console**.
3. Le shell s'ouvre avec utilisateur par défaut `root` directement.
### 2. Installation du serveur SSH
Depuis la console du conteneur entrer les commandes suivantes pour installer le service SSH (fichier binaire `sshd`) et les dépendances :
```sh
apt update 
apt install openssh-server -y
```
### 3. Vérifier (et démarrer) le service SSH

```sh
service ssh start
```

---
# Attribuer un mot de passe root
Souvent dans Docker, le compte `root` n’a pas de mot de passe, pour éviter des erreurs et des problèmes avec ssh et ansible alors on modifie cela. Pour en mettre un :
```sh
passwd root
```

Il nous sera demandé de saisir puis confirmer un mot de passe.
## Créer un un autre utilisateur pour Ansible
### Pourquoi ?
Le fait d’autoriser une connexion SSH directe en tant que **root** est généralement considéré comme moins sécurisé pour plusieurs raisons :
- **Surface d’attaque plus large** : S’il est compromis l’attaquant dispose immédiatement de tous les droits.
- **Cible évidente** : brute force sur utilisateur "root".
- **Moins de traçabilité** : Dans le journal des logs seul l'utilisateur `root` est utilisé, ce qui ne permet pas d'identifier qui a effectuer quelles actions.
### Créer un utilisateur standard avec droits sudo
```sh
adduser ansible
usermod -aG sudo ansible
```

### Désactiver l’authentification root par mot de passe
Dans etc/ssh/sshd_config dé-commenter la ligne suivante et la modifier pour **interdire complètement** la connexion SSH directe de root (même via une clé) :
```
PermitRootLogin no
```

Cela force l’utilisation d’un autre utilisateur pour la connexion SSH, ou impose l’usage d’une clé SSH s’il faut malgré tout se connecter en root.

Redémarrer le service :
```sh
service ssh restart
```

---
---
---
## ==Info Samy ==
Hello, pour automatiser le lancement de certain services dans un conteneur GNS3 vous pouvez utiliser le fichier `/gns3/run-cmd.sh`, ce fichier est lancé automatiquement au démarrage d'un conteneur, à l'intérieur vous avez par défaut :

```
#!/bin/sh 
# run docker startup, first arg is new PATH, remainder is command  PATH="$1" shift exec "$@"
```

Vous pouvez ajouter entre `shift` et `exec "$@"` vos commandes, par exemple ici pour mon conteneur `DMZ-DNS` je veux démarrer automatiquement SSH et BIND :

```
#!/bin/sh 
# run docker startup, first arg is new PATH, remainder is command  PATH="$1" shift  echo "Starting services..." # Démarrer BIND en arrière plan : nohup /usr/sbin/named -4 -u bind -g > /var/log/named.log 2>&1 & # Démarrer le serveur SSH : service ssh start &  exec "$@"
```


Pour rendre persistant un conteneur : 
-> Paramètres -> Advanced -> "Additionnal directories to make persistent..." : 
```
/bin
/boot
/etc
/gns3
/home
/lib
/lib64
/media
/mnt
/opt
/root
/run
/sbin
/srv
/tmp
/usr
/var
```
Ajoutez tout ça et votre conteneur ne perdra plus rien (à priori)
M-K89y~`Ng*Xnwt4j\Fk

En SSH sur les containers j'ai un bash semi-interactif "PTY allocation request failed on channel 0" j'ai testé sur un nouveau container tout neuf et j'ai pas ce problème. 
J'ai l'impression que le problème semble venir du fait que l'on est rendu persistant tout le système de fichiers. 
Si quelqu'un a une piste je suis preneur.
@everyone
