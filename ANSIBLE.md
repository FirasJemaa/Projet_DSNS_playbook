## Prérequis
1. [[Configuration pour Ansible]] 
2. [[PKI]]
---
Voici une proposition de démarche étape par étape pour :
1. Mettre en place et configurer Ansible sur **IT-ANSIBLE**.
2. Automatiser le déploiement et la configuration des serveurs.
3. Sécuriser au maximum les services (par exemple : SMTPS, HTTPS, etc.).

L’idée est de te donner un fil directeur, tout en expliquant les concepts et bonnes pratiques. Tu pourras adapter chaque étape selon tes besoins précis et ta propre organisation.

---
# Préparation d’IT-ANSIBLE
## Mettre la carte réseau en DHCP
1. **Ajouter le poste au switch "IT-SW"** dans GNS3.
2. **Configurer le poste manuellement :**
    - **Clic droit sur le poste** dans GNS3 → **Configure**.
    - Aller dans l’onglet **General settings**.
    - Descendre jusqu’à **Network configuration** et cliquer sur **Edit**.
3. Modification du fichier de configuration réseau en DHCP en dé commentant les lignes suivantes :
```sh
auto eth0
iface eth0 inet dhcp
	hostname IT-ANSIBLE
```
## 1. Installation d’Ansible
### 1.1 Installation des paquets : 
Sur ton serveur Debian (it-ansible.itway.local), installe Ansible :
```bash
apt update
apt install ansible sshpass python3-apt -y
```

- `sshpass` pour les tests au départ en utilisant l’authentification par mot de passe, ensuite passer à l’authentification par clé SSH.
- `python3-apt` est nécessaire pour gérer des paquets sur des distributions Debian/Ubuntu via Ansible.
### 1.2 Génération d’une clé SSH :
Pour sécuriser et simplifier les connexions SSH aux cibles, je génère une clé ED25519 sur **IT-ANSIBLE** :
```bash
ssh-keygen -t ed25519
```

 Je laisse le **chemin par défaut** (`root/.ssh/id_ed25519`) et je protège avec une **passphrase** `3pxG$fyYGAG`.

### 1.3 Diffusion de la clé publique : 
Il faut copié la clé publique sur les différentes machines :
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub <user>@<Hostname>
```

On remplace :  
- `user` par l’utilisateur privilégié (`ansible`).
- `dmz-smtp` : <span style="color:orange">temporairement</span> vu que le **DNS interne** n'est pas terminé, je mes l'adresse IP de la machine.
#### 1.3.1 <span style="color:orange">Problème rencontré</span>
Lorsque j’exécute la commande, j'ai l'erreur suivante :
```sh
/usr/bin/ssh-copy-id: ERROR: ssh: connect to host 10.10.10.2 port 22: No route to host
```

Je me rencontre que je n'arrive pas à atteindre les machines dans la **DMZ**. Donc j'essaye de faire un ping vers le routeur principal (**Border-RT**), le ping passe. Et lorsque j'essaye de ping le routeur dmz (**DMZ-RT**) le ping ne passe plus : 
```sh
From 212.30.97.108 icmp_seq=1 Packet filtered
```

Vu que l'adresse IP n'existe pas dans mon réseaux cela veut dire que le routeur principal envoie l'adresse IP à l’extérieur du réseaux. Donc, il faut essayé de créé une route pour éviter ce problème : 
1. Connexion à l'interface du routeur (**BORDER-RT**)
2. Je vais dans : Configuration > Réseau > Routage.
3. J'ajoute une route statique : 
	- Destination : `DMZ-INT` (Network que j'ai créé dans les objets)
	- Interface : `DMZ`
	- Address range : `10.10.10.0/28` (qui se rempli automatiquement lorsqu'on sélectionne les deux champ d'avant)
	- Gateway : `GW-DMZ` (J'ai créé cette objet dans Host avec la bonne passerelle `10.11.11.2`)

![[Pasted image 20250322171428.png]]

Je teste avec un ping que ça fonctionne correctement est c'est bon.
#### 1.3.2 Test de la connexion sans mot de passe
Je lance la commande :
```
ssh ansible@10.10.10.2
```
### 1.4 [[Configuration SSH]] :
Dans `~/.ssh/config` sur IT-ANSIBLE), j'ajoute les hôtes Ansible pour simplifier la connexion :

```text
Host dmz-smtp
  HostName <Adresse IP>
  User ansible
  IdentityFile ~/.ssh/id_ed25519
```

Je fais de même pour DMZ-DNS, DMZ-RPROXY, DMZ-WEB.

---
## 1.2 Structure de l’arborescence Ansible
Pour garder une organisation claire, j'utilise la hiérarchie suivante :
```
/etc/ansible/
├── ansible.cfg
├── inventory/
│   ├── hosts.ini
│   └── group_vars/
│       ├── dmz-smtp.yml
│       ├── dmz-dns.yml
│       ├── dmz-rproxy.yml
│       ├── dmz-web.yml
│       ├── srv-ads01.yml
│       ├── srv-pki.yml
│       ├── srv-mail.yml
│       ├── srv-intranet.yml
│       └── it-grafana-prometheus.yml
├── roles/
│   ├── dmz-smtp/
│   │   ├── tasks/
│   │   ├── handlers/
│   │   ├── templates/
│   │   └── files/
│   ├── dmz-dns/
│   │   ├── tasks/
│   │   ├── handlers/
│   │   ├── templates/
│   │   └── files/
│   ├── dmz-rproxy/
│   │   ├── tasks/
│   │   ├── handlers/
│   │   ├── templates/
│   │   └── files/
│   ├── dmz-web/
│   │   ├── tasks/
│   │   ├── handlers/
│   │   ├── templates/
│   │   └── files/
│   ├── srv-ads01/     
│   │   ├── tasks/
│   │   ├── handlers/
│   │   ├── templates/
│   │   └── files/
│   ├── srv-pki/     
│   │   ├── tasks/
│   │   ├── handlers/
│   │   ├── templates/
│   │   └── files/
│   ├── srv-mail/ 
│   │   ├── tasks/
│   │   ├── handlers/
│   │   ├── templates/
│   │   └── files/
│   ├── srv-intranet/ 
│   │   ├── tasks/
│   │   ├── handlers/
│   │   ├── templates/
│   │   └── files/
│   └── it-grafana/
│       ├── tasks/
│       ├── handlers/
│       ├── templates/
│       └── files/
└── playbooks/
    ├── setup-smtp.yml
    ├── setup-dmz.yml
    ├── setup-web.yml
    ├── setup-dns.yml
    ├── setup-ad.yml
    ├── setup-pki.yml
    ├── setup-mail.yml
    ├── setup-intranet.yml
    └── setup-grafana-prometheus.yml
```

- **`ansible.cfg`** : Fichier de configuration global d’Ansible (ex : répertoires par défaut, etc.).
- **`inventory/hosts.ini`** : Liste des hôtes et groupes.
- **`group_vars/`** : Variables propres à chaque groupe d’hôtes (ou hôte individuel).
- **`roles/`** : Rôles Ansible contenant les tâches, templates, fichiers de config, etc.

Contenu de `hosts.ini` :
```ini
[dmz-smtp]
dmz-smtp

[dmz-dns]
dmz-dns

[dmz-web]
dmz-web

[dmz-rproxy]
dmz-rproxy

...

[dmz:children]
dmz-smtp
dmz-dns
dmz-web
dmz-rproxy
```

---
---
---
---
Voici un **cours** (ou guide) étape par étape pour comprendre à quoi sert chacun des éléments de l’arborescence Ansible, et **pourquoi** on les organise ainsi. Nous prendrons comme exemple l’arborescence que vous avez donnée :

```
/etc/ansible/
├── ansible.cfg
├── inventory/
│   ├── hosts.ini
│   └── group_vars/
│       ├── dmz-smtp.yml
│       ├── dmz-dns.yml
│       ├── dmz-rproxy.yml
│       └── dmz-web.yml
├── roles/
│   ├── dmz-smtp/
│   ├── dmz-dns/
│   ├── dmz-rproxy/
│   └── dmz-web/
└── playbooks/
    ├── setup-smtp.yml
    ├── setup-dmz.yml
    ├── setup-web.yml
    └── setup-dns.yml
```

Nous allons passer en revue chaque **dossier** et **fichier**, en expliquant :
1. **À quoi il sert** dans la logique Ansible.    
2. **Comment** et **pourquoi** on l’utilise.    

---

# 1. Dossier racine : `/etc/ansible/`

C’est le **point d’entrée** par défaut d’Ansible lorsqu’on l’installe sur un système Linux (Debian, Ubuntu, CentOS, etc.). Vous pouvez tout à fait déplacer cette arborescence ailleurs (par exemple, un dépôt Git local), mais `/etc/ansible/` reste un choix classique.

---

# 2. `ansible.cfg`

**C’est le fichier de configuration global d’Ansible.**
- **Son rôle** : définir le comportement d’Ansible lors de l’exécution des playbooks.
    - Exemples de paramètres :
        - `inventory = inventory/hosts.ini` pour spécifier l’inventaire par défaut.
        - `remote_user = ansible` pour préciser l’utilisateur SSH par défaut (si besoin).
        - `host_key_checking = False` si on ne souhaite pas vérifier la clé SSH de l’hôte distant, etc.
- **Pourquoi est-ce important ?**
    - Cela vous évite de renseigner ces informations à chaque commande,        
    - Vous pouvez définir des options globales (ex. chemin du répertoire des rôles, appels de plugins, callback, etc.).

Dans de nombreux cas, on laisse la configuration par défaut et on ne modifie que quelques champs clés (inventaire, utilisateur SSH, etc.).

---

# 3. Dossier `inventory/`
Dans Ansible, **l’inventaire** décrit les machines (ou groupes de machines) sur lesquelles on va agir. C’est là qu’on déclare l’adresse IP ou le FQDN de chaque hôte, ainsi que des informations de connexion.
## 3.1 `hosts.ini`

- **Fichier d’inventaire principal** (format `.ini` ou YAML selon vos préférences).
    
- **Contenu typique** :
```ini
[dmz-smtp]
dmz-smtp.itway.local ansible_host=192.168.20.10

[dmz-dns]
dmz-dns.itway.local ansible_host=192.168.20.11

[dmz-rproxy]
dmz-rproxy.itway.local ansible_host=192.168.20.12

[dmz-web]
dmz-web.itway.local ansible_host=192.168.20.13
```
    
Vous pouvez avoir plusieurs groupes (ex. `[dmz-smtp]`, `[dmz-web]`, etc.). Quand on exécute un playbook ciblant un groupe particulier, Ansible s’applique sur tous les hôtes du groupe.
    
- **Pourquoi on le sépare dans un répertoire `inventory/` ?**
    - Pour mieux organiser et éventuellement créer plusieurs inventaires (production, pré-production, tests, etc.).

## 3.2 `group_vars/`

- **But** : stocker les **variables** propres à un groupe d’hôtes (ou un hôte si vous le souhaitez).
    
- **Pourquoi ?**
    - Certains paramètres sont communs à tous les serveurs d’un même groupe. Par exemple, pour `dmz-smtp`, on peut y définir des paramètres Postfix, des chemins de configuration, etc.

- **Exemple** : dans `group_vars/dmz-smtp.yml`, vous pourriez mettre :    
    ```yaml
    postfix_config_path: "/etc/postfix"
    postfix_domain_name: "mail.itway.fr"
    spamassassin_enable: true
    ```
    
    Ainsi, toutes les machines du groupe `[dmz-smtp]` auront accès à ces variables.
    
- **Comment Ansible les utilise ?**
    
    - Lorsqu’on exécute un playbook sur un groupe, Ansible charge automatiquement ce fichier de variables.
        

---

# 4. Dossier `roles/`

Les **rôles** sont la méthode recommandée par Ansible pour organiser et réutiliser votre code. Au lieu d’écrire un gros playbook unique, on découpe par fonctionnalité (rôle).

## 4.1 Pourquoi des rôles ?

- **Lisibilité** : On sépare la configuration SMTP, DNS, Reverse Proxy, etc., dans des rôles distincts.
    
- **Réutilisabilité** : Si vous devez redeployer le même service sur un autre hôte, vous pouvez simplement réutiliser le même rôle.
    
- **Arborescence standard** : Ansible sait, grâce à une arborescence précise, où chercher les tâches, les templates, les handlers…
    

## 4.2 Structure d’un rôle

Chaque dossier (par ex. `dmz-smtp/`) contient souvent :

1. **`tasks/main.yml`** : la liste principale de tâches à exécuter pour installer/configurer le service.
    
2. **`handlers/main.yml`** : les handlers sont des actions déclenchées après qu’une tâche notifie un changement (ex. redémarrer Postfix après mise à jour de la conf).
    
3. **`templates/`** : fichiers `Jinja2` (ex. `postfix.conf.j2`) qui seront copiés sur la machine cible.
    
4. **`files/`** : fichiers statiques, éventuellement nécessaires (script shell, certificat, etc.).
    
5. **`vars/` ou `defaults/`** (facultatif) : variables spécifiques à ce rôle.
    

### Exemple : `roles/dmz-smtp/tasks/main.yml`

```yaml
---
- name: Installer Postfix et SpamAssassin
  apt:
    name:
      - postfix
      - spamassassin
    state: present
    update_cache: yes

- name: Copier la configuration postfix
  template:
    src: postfix.conf.j2
    dest: "{{ postfix_config_path }}/main.cf"
  notify:
    - restart postfix
```

### Exemple : `roles/dmz-smtp/handlers/main.yml`

```yaml
---
- name: restart postfix
  service:
    name: postfix
    state: restarted
```

- **Pourquoi des handlers ?**
    
    - Éviter de redémarrer un service à chaque tâche. Les handlers ne sont déclenchés qu’une fois, à la fin du play, **si** la configuration a changé.

---
# 5. Dossier `playbooks/`
C’est le répertoire qui **centralise vos playbooks** Ansible. Un playbook est un fichier YAML qui liste :
1. Les **hôtes** cibles (un groupe d’hôtes, p. ex. `[dmz-smtp]`).
2. Les **rôles** ou **tâches** à exécuter sur ces hôtes.   
## 5.1 Exemple : `setup-smtp.yml`

```yaml
---
- name: Déploiement du serveur SMTP dans la DMZ
  hosts: dmz-smtp
  become: yes
  roles:
    - dmz-smtp
```

**Explications** :
- **`hosts: dmz-smtp`** signifie qu’on applique ce playbook sur tous les hôtes définis dans la section `[dmz-smtp]` du fichier `hosts.ini`.
- **`become: yes`** indique qu’on exécute les tâches en privilège `root` (sudo).
- **`roles: - dmz-smtp`** charge le rôle `dmz-smtp` (dans `/etc/ansible/roles/dmz-smtp`) et exécute les tâches de `tasks/main.yml`, les handlers, etc.   

## 5.2 Les autres playbooks
- `setup-dmz.yml` : Peut-être un playbook **global** pour déployer toutes les machines de la DMZ (SMTP, DNS, Reverse Proxy, Web) en série.    
- `setup-web.yml` : Déploie le rôle `dmz-web`.
- `setup-dns.yml` : Déploie le rôle `dmz-dns`.    

Grâce à ces playbooks, vous avez deux approches possibles :
1. **Approche ciblée** :    
    - `ansible-playbook setup-smtp.yml` (pour configurer uniquement SMTP), etc.        
2. **Approche globale** :    
    - `ansible-playbook setup-dmz.yml` (pour configurer **tous** les rôles de la DMZ d’un coup).        

---
# 6. Pourquoi cette organisation ?

1. **Lisibilité** : En regardant l’arborescence, on repère facilement où se trouve la config SMTP, DNS, Web, etc.
    
2. **Maintenance facilitée** :
    
    - Si vous devez changer la conf SMTP, vous ouvrez `roles/dmz-smtp/`.
        
    - Si vous devez ajouter une variable commune à la DMZ, vous allez dans `group_vars/dmz-smtp.yml` (ou `dmz-dns.yml`, etc. selon le service).
        
3. **Réutilisation** : Votre rôle `dmz-smtp` pourrait être utilisé dans un autre environnement, juste en modifiant l’inventaire.
    
4. **Évolutivité** : Vous pouvez ajouter des rôles (ex. `srv-ads01`, `srv-pki`, `srv-mail`, `it-graphana`) de la même manière, sans casser la structure existante.
    

---

# 7. Résumé final

- **`ansible.cfg`** : Configuration générale d’Ansible (chemin de l’inventaire, politiques SSH, etc.).
    
- **`inventory/`** : Décrit sur quels serveurs on va agir et regroupe les variables de groupe/hôte.
    
    - **`hosts.ini`** : Liste les hôtes et groupes (ex. `[dmz-smtp]`, `[dmz-web]`, etc.).
        
    - **`group_vars/`** : Variables spécifiques à un groupe d’hôtes (ex. `dmz-smtp.yml`).
        
- **`roles/`** : Chaque dossier représente un **service** à installer ou configurer (DMZ SMTP, DNS, RProxy, Web).
    
    - **`tasks/main.yml`** : Tâches principales du rôle.
        
    - **`handlers/main.yml`** : Actions de notification (ex. redémarrer un service).
        
    - **`templates/`** et **`files/`** : Fichiers de configuration (Jinja2 ou statiques).
        
- **`playbooks/`** : Point d’entrée du déploiement. Chaque playbook appelle un ou plusieurs rôles sur un ou plusieurs groupes d’hôtes.
    
    - `setup-smtp.yml` : Lance le rôle `dmz-smtp` sur `[dmz-smtp]`.
        
    - `setup-dmz.yml` : Lance tous les rôles DMZ (smtp, dns, etc.) sur tous les hôtes DMZ, etc.
        

**Conclusion** :  
Cette structure est celle recommandée par la documentation Ansible pour des projets de taille moyenne ou grande, car elle vous permet d’avoir une séparation claire entre :

- Les **inventaires** (où et sur quoi déployer),
    
- Les **rôles** (ce qu’on déploie et comment),
    
- Les **variables** (valeurs de configuration),
    
- Les **playbooks** (l’enchaînement concret de ce qu’on va faire).
    

Vous bénéficiez ainsi d’un environnement organisé, évolutif et maintenable.
