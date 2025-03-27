# SERVEUR PKI - Déploiement Automatisé avec Ansible


## 1. Organisation des rôles et fichiers Ansible

```
├── ansible.cfg
├── inventory/
│   └── hosts
├── playbooks/
│   └── pki.yml
└── roles/
    └── srv-pki/
        ├── files/
        ├── handlers/
        │   └── main.yml
        ├── tasks/
        │   └── main.yml
        ├── templates/
        │   └── openssl.cnf.j2
        └── vars/
            └── main.yml
```

**Commentaires :**
- Cette structure suit les best practices d'Ansible
- Le rôle est auto-contenu avec séparation claire des variables, templates et tâches
- Il faudra ajouter les configurations respectives dans chaque fichier avec la commande `nano srv-pki/`
  
## 2. Configuration `inventory/hosts.ini`

```ini
[srv-pki]
srv-pki.itway.local ansible_host=172.16.50.3 ansible_user=ansible ansible_become=true ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

**Commentaires :**
- L'utilisation d'un FQDN est recommandée pour les PKI
- L’authentification SSH par clé **est déjà en place**, ce qui assure une connexion sécurisée
## 3. Variables pour la PKI `roles/pki/vars/main.yml`

```yaml
# Parametres de base PKI
pki_country: "FR"
pki_state: "Ile-de-France"
pki_locality: "Paris"
pki_organization: "ITWay"
pki_organizational_unit: "ITDepartment"
pki_common_name: "srv-pki.itway.local"
pki_email_address: "admin@itway.fr"

# Configuration des chemins
pki_ca_dir: "/etc/pki"
pki_ca_key_name: "ca.key"
pki_ca_cert_name: "ca.crt"
pki_openssl_conf: "/etc/ssl/openssl.cnf"

# Gestion des index/crl
pki_index_file: "index.txt"
pki_serial_file: "serial"
pki_crl_number_file: "crlnumber"

# Parametres techniques (ajoutes)
pki_key_bits: 4096
pki_ca_days: 3650
pki_default_md: "sha256"
pki_crl_days: 30

# Permissions (ajoutees)
pki_private_dir_mode: "0700"
pki_key_mode: "0600"
pki_cert_mode: "0644"
```
**Commentaires :**
- Tous les paramètres techniques sont maintenant externalisés en variables
- Ajout de paramètres de sécurité (permissions, algorithmes)
- La durée de validité est configurable (10 ans par défaut pour la CA)

## 4. Configuration OpenSSL 'templates/openssl.cnf.j2`

```jinja2
[ ca ]
default_ca = CA_default

[ CA_default ]
dir             = {{ pki_ca_dir }}
certs           = $dir/certs
crl_dir         = $dir/crl
new_certs_dir   = $dir/newcerts
database        = $dir/{{ pki_index_file }}
serial          = $dir/{{ pki_serial_file }}
RANDFILE        = $dir/private/.rand
private_key     = $dir/private/{{ pki_ca_key_name }}
certificate     = $dir/certs/{{ pki_ca_cert_name }}
default_md      = {{ pki_default_md }}
policy          = policy_match
email_in_dn     = no
default_days    = {{ pki_ca_days }}
default_crl_days = {{ pki_crl_days }}
crl_days        = {{ pki_crl_days }}
unique_subject  = no
x509_extensions = v3_ca  # <-- indique quelle section utiliser lors de la creation

[ policy_match ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ v3_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_server ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:false
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[ v3_client ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:false
keyUsage = critical, digitalSignature
extendedKeyUsage = clientAuth
```

**Commentaires :**
- Utilisation des variables pour tous les paramètres dynamiques
- Ajout de sections dédiées pour les certificats serveur et client
- Meilleure gestion des contraintes X509v3

## 5. Configuration tâches du rôle PKI `roles/pki/tasks/main.yml`
```
# roles/pki/tasks/main.yml

- name: Installer les paquets necessaires
  apt:
    name:
      - openssl
      - ca-certificates
    state: present
    update_cache: yes
  tags: installation
  notify: restart openssl

- name: Creer l'arborescence PKI
  file:
    path: "{{ item.path }}"
    state: directory
    mode: "{{ item.mode }}"
  loop:
    - { path: "{{ pki_ca_dir }}", mode: "0755" }
    - { path: "{{ pki_ca_dir }}/certs", mode: "0755" }
    - { path: "{{ pki_ca_dir }}/crl", mode: "0755" }
    - { path: "{{ pki_ca_dir }}/newcerts", mode: "0755" }
    - { path: "{{ pki_ca_dir }}/private", mode: "{{ pki_private_dir_mode }}" }
  tags: setup

- name: Initialiser les fichiers de base
  file:
    path: "{{ pki_ca_dir }}/{{ item.file }}"
    state: touch
    mode: "{{ item.mode }}"
  loop:
    - { file: "{{ pki_index_file }}", mode: "0644" }
    - { file: "serial", mode: "0644" }
    - { file: "crlnumber", mode: "0644" }
  tags: setup

- name: Verifier si le fichier serial existe
  stat:
    path: "{{ pki_ca_dir }}/serial"
  register: serial_file
  tags: setup

- name: Initialiser le numero de serie
  copy:
    dest: "{{ pki_ca_dir }}/serial"
    content: "01"
    mode: "0644"
  when: not serial_file.stat.exists
  tags: setup

- name: Verifier si le fichier crlnumber existe
  stat:
    path: "{{ pki_ca_dir }}/crlnumber"
  register: crl_file
  tags: setup

- name: Initialiser le numero de CRL
  copy:
    dest: "{{ pki_ca_dir }}/crlnumber"
    content: "01"
    mode: "0644"
  when: not crl_file.stat.exists
  tags: setup

- name: Verifier si le fichier index.txt existe
  stat:
    path: "{{ pki_ca_dir }}/{{ pki_index_file }}"
  register: index_file
  tags: setup

- name: Creer le fichier index.txt vide
  copy:
    dest: "{{ pki_ca_dir }}/{{ pki_index_file }}"
    content: ""
    mode: "0644"
  when: not index_file.stat.exists
  tags: setup

- name: Deployer le fichier de configuration OpenSSL
  template:
    src: openssl.cnf.j2
    dest: "{{ pki_openssl_conf }}"
    owner: root
    group: root
    mode: "0644"
  tags: configuration

- name: Generer le fichier aleatoire OpenSSL
  command: openssl rand -out {{ pki_ca_dir }}/private/.rand 4096
  args:
    creates: "{{ pki_ca_dir }}/private/.rand"
  tags: setup

- name: Securiser les permissions des dossiers
  file:
    path: "{{ item }}"
    mode: "0750"
    owner: root
    group: root
  loop:
    - "{{ pki_ca_dir }}"
    - "{{ pki_ca_dir }}/private"
  tags: security

# ==================== #
# PARTIE CERTIFICATS CA #
# ==================== #

- name: Generer la cle privee de la CA
  command: >
    openssl genrsa
    -out {{ pki_ca_dir }}/private/{{ pki_ca_key_name }}
    {{ pki_key_bits }}
  args:
    creates: "{{ pki_ca_dir }}/private/{{ pki_ca_key_name }}"
  notify: Secure CA private key
  tags: certificates

- name: Generer le certificat auto-signe de la CA
  command: >
    openssl req -x509 -new -nodes
    -key {{ pki_ca_dir }}/private/{{ pki_ca_key_name }}
    -sha256
    -days {{ pki_ca_days }}
    -config {{ pki_openssl_conf }}
    -extensions v3_ca
    -subj "/C={{ pki_country }}/ST={{ pki_state }}/L={{ pki_locality }}/O={{ pki_organization }}/OU={{ pki_organizational_unit }}/CN={{ pki_common_name }}/emailAddress={{ pki_email_address }}"
    -out {{ pki_ca_dir }}/certs/{{ pki_ca_cert_name }}
  args:
    creates: "{{ pki_ca_dir }}/certs/{{ pki_ca_cert_name }}"
  tags: certificates

- name: Valider le certificat CA
  command: openssl x509 -in {{ pki_ca_dir }}/certs/{{ pki_ca_cert_name }} -noout -text
  register: cert_check
  changed_when: false
  tags: validation

- name: Afficher les infos du certificat
  debug:
    msg: |
      CA generee avec succes !
      Validite: {{ cert_check.stdout | regex_search('Not After : (.+)') }}
      Sujet: {{ cert_check.stdout | regex_search('Subject: (.+)') }}
  when: cert_check is defined
  tags: validation

# ================== #
# GESTION DES CRL    #
# ================== #

- name: Generer le CRL initial
  command: >
    openssl ca -gencrl
    -config {{ pki_openssl_conf }}
    -out {{ pki_ca_dir }}/crl/crl.pem
  args:
    creates: "{{ pki_ca_dir }}/crl/crl.pem"
  tags: crl

- name: Verifier le CRL
  command: openssl crl -in {{ pki_ca_dir }}/crl/crl.pem -noout -text
  register: crl_check
  changed_when: false
  when: crl_check is defined
  tags: validation
```

## Fichier handler associé ```roles/pki/handlers/main.yml``` :
```
- name: Secure CA private key
  file:
    path: "{{ pki_ca_dir }}/private/{{ pki_ca_key_name }}"
    mode: "{{ pki_key_mode | default('0600') }}"
    owner: root
    group: root
  #comment: "Sécurise la clé privée après sa création"

- name: restart openssl
  service:
    name: openssl
    state: restarted
  when: false  # Désactivé car pas de service dédié
  #comment: "Exemple pour d'autres services qui utiliseraient OpenSSL"
```
## 6. Playbook principal `playbooks/pki.yml`
```
# playbooks/pki.yml
---
- name: Deploiement complet du serveur PKI
  hosts: srv-pki
  become: yes
  gather_facts: yes
  
  pre_tasks:
    - name: Verifier que le systeme est compatible
      assert:
        that:
          - ansible_os_family == "Debian"
          - ansible_distribution == "Ubuntu"
          - ansible_distribution_major_version | int >= 22
        msg: "Ce playbook nécessite Ubuntu 22 ou supérieur"

  roles:
    - srv-pki

  post_tasks:
    - name: Afficher les informations du certificat
      debug:
        msg: "CA déployee avec succès. Certificat valide jusqu'au {{ cert_check.stdout | regex_search('Not After : (.+)') }}"
      when: cert_check is defined
```

Commentaires :

Ajout de pré-vérifications système

Affichage des informations de validation en fin de déploiement

Meilleure gestion des erreurs

## 7. Points de sécurité et bonnes pratiques 
### Sauvegarde des clés :

- La clé privée de la CA doit être sauvegardée hors ligne car c'est l'élément le plus critique de notre PKI (par exemple sur un support physique de type clé USB chiffrée).

- Il est également possible d'tiliser Ansible Vault qui est intégré à ansible, afin de chiffrer nos données sensibles dans nos fichiers yaml.

### Rotation des certificats :

```
- name: Renouveler le certificat CA avant expiration
  command: >
    openssl x509 -x509toreq
    -in {{ pki_ca_dir }}/certs/{{ pki_ca_cert_name }}
    -signkey {{ pki_ca_dir }}/private/{{ pki_ca_key_name }}
    -out {{ pki_ca_dir }}/private/ca_renew.csr
  when: cert_check.stdout | regex_search('Not After : (.+)') | expiration_date_compare
```

Révocation :

```
- name: Révoquer un certificat
  command: >
    openssl ca -revoke {{ pki_ca_dir }}/newcerts/{{ cert_to_revoke }}.pem
    -config {{ pki_openssl_conf }}
  vars:
    cert_to_revoke: "01"  # Numéro de série du certificat
```

Journalisation :

Ajouter des tâches pour logger les actions importantes

## 8. Exemple de signature de certificat serveur 
```
- name: Generer une CSR pour un serveur
  command: >
    openssl req -new -newkey rsa:2048
    -nodes -keyout {{ pki_ca_dir }}/private/server.key
    -out {{ pki_ca_dir }}/csr/server.csr
    -subj "/CN=server.example.com/O=My Company"

- name: Signer le certificat serveur
  command: >
    openssl ca -in {{ pki_ca_dir }}/csr/server.csr
    -out {{ pki_ca_dir }}/certs/server.crt
    -config {{ pki_openssl_conf }}
    -extensions v3_server
```

Commentaires finaux :

## 9. Pour le déployer

```
ansible-playbook -i inventory/hosts playbooks/pki.yml --ask-vault-pass
```

## 10. Erreurs rencontrées : 
```
ERROR! Unexpected Exception, this is probably a bug: 'utf-8' codec can't encode character '\udcc3' in position 346: surrogates not allowed
```
Cause probable :
Un caractère spécial mal encodé (é, à, ç, etc.)

Un fichier YAML enregistré en latin1 ou Windows-1252 au lieu d’UTF-8

Un copier-coller depuis un éditeur non UTF-8

Solution :

Identifier le fichier fautif avec un mode verbeux :
```
ansible-playbook -i inventory/hosts.ini playbooks/setup-pki.yaml -vvv
```
Vérifier l'encodage du fichier suspect :
```
file playbooks/setup-pki.yaml
```
S’il n’est pas en UTF-8, le convertir :
```
iconv -f ISO-8859-1 -t UTF-8 playbooks/setup-pki.yaml -o playbooks/setup-pki-fixed.yaml
mv playbooks/setup-pki-fixed.yaml playbooks/setup-pki.yaml
```
Corriger tous les fichiers YAML en une commande :
```
find . -name "*.yml" -exec bash -c 'iconv -f ISO-8859-1 -t UTF-8 "{}" -o "{}.utf8"
```

## 10. Problèmes rencontrés et Résolution 

### Contexte initial

- L’objectif était de déployer un rôle Ansible (`srv-pki`) sur un serveur distant (`srv-pki.itway.local`) en utilisant l’utilisateur `ansible` (non-root).
- L’accès SSH par clé privée était déjà en place et fonctionnel.
- Le rôle Ansible nécessitait des tâches avec privilèges (`apt`, fichiers dans `/etc/`, etc.), donc une élévation de privilèges (`sudo`) était nécessaire.

---

## Problèmes rencontrés

### 1. Erreur : `Missing sudo password`

```bash
fatal: [srv-pki.itway.local]: FAILED! => {"msg": "Missing sudo password"}
```

Ansible tentait d’utiliser `sudo` via `become: true`, mais :

- Aucun mot de passe `sudo` n’était fourni
    
- L’utilisateur `ansible` n’était pas autorisé à utiliser `sudo` sans mot de passe
    

---

### 2. Erreur : `sudo must be owned by uid 0 and have the setuid bit set`

```bash
sudo: /usr/bin/sudo must be owned by uid 0 and have the setuid bit set
```

Sur la machine distante, le binaire `sudo` avait des permissions corrompues :

- Il n'était pas possédé par root
    
- Le "bit setuid" n’était pas activé
    

Conséquence : `sudo` était inutilisable, même avec les bons droits d’utilisateur.

---

### 3. Tentatives sans succès

- Suppression de `become: true` : entraînait des erreurs de permissions sur les tâches sensibles (`apt`, `/etc`, `/var`)
    
- Ajout de `ansible_become=false` : même limitation
    
- `--ask-become-pass` : bloquant en environnement automatisé
    
- Passage en `ansible_user=root` : contraire aux objectifs (utiliser un utilisateur dédié)
    

---

## Corrections apportées

### 1. Réparation du binaire `sudo` sur la machine distante

Sur la machine `srv-pki.itway.local`, en root :

```bash
chown root:root /usr/bin/sudo
chmod 4755 /usr/bin/sudo
```

Cela a permis de :

- Restaurer la propriété correcte (`root`)
    
- Activer le setuid (`chmod 4755`) pour permettre l’élévation
    

---

### 2. Ajout de l'utilisateur `ansible` au fichier sudoers (via visudo)

Commande :

```bash
visudo
```

Ajout de la ligne suivante :

```bash
ansible ALL=(ALL) NOPASSWD: ALL
```

Cela a permis à Ansible d’utiliser `sudo` sans mot de passe, essentiel pour le fonctionnement automatique d’Ansible avec `become: true`.

---

### 3. Mise à jour de l’inventaire Ansible

Fichier `inventory/hosts.ini` :

```ini
[srv-pki]
srv-pki.itway.local ansible_host=172.16.50.3 ansible_user=ansible ansible_ssh_private_key_file=~/.ssh/id_ed25519 ansible_become=true ansible_become_method=sudo ansible_become_user=root
```

---

### 4. Configuration du playbook Ansible

Fichier `playbooks/setup-pki.yaml` :

```yaml
- name: Deploiement complet du serveur PKI
  hosts: srv-pki
  become: true
  gather_facts: yes

  roles:
    - srv-pki
```

---

## Résultat final

✔️ Le rôle `srv-pki` s'exécute désormais correctement via Ansible avec l’utilisateur `ansible`  
✔️ Aucune demande de mot de passe sudo  
✔️ Toutes les tâches nécessitant une élévation de privilèges fonctionnent (`apt`, fichiers système, certificats)  
✔️ L’architecture respecte les bonnes pratiques : **utilisateur dédié + `sudo` sécurisé**

---

## Recommandations pour l’avenir

- Toujours valider que `sudo` est correctement installé et configuré
    
- Ne pas désactiver `become` si des tâches système sont présentes
    
- Vérifier que le binaire `/usr/bin/sudo` est bien :
    
    ```bash
    -rwsr-xr-x 1 root root ...
    ```
    
- Utiliser Ansible Vault pour sécuriser les données sensibles si besoin
    

---
