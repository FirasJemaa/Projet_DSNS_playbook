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

**Commentaires :**

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

---
## 10. Problèmes rencontrés et résolution
### 1. Erreur d’encodage YAML / UTF-8
### Erreur :
```sh
Syntax Error while loading YAML.
  . unacceptable character #x0083: control characters are not allowed
  in "<unicode string>", position 501
```

>[!Causes probable]
>- Caractère spécial mal encodé (`é`, `à`, `ç`, etc.)
>- Copier-coller depuis un éditeur non UTF-8
>- Caractère invisible dans un fichier texte (`\x83`, `\udcXX`…)

### Solution :
1. Identifier le fichier fautif :
```bash
ansible-playbook -i inventory/hosts.ini playbooks/setup-pki.yaml -vvv
```
2. Rechercher les caractères suspects (exemple avec `\x83`) :
```bash
grep -rl $'\x83' .
```
3. Corriger les erreurs
4. Convertir tous les `.yml` en UTF-8 (en masse) :
```bash
find . -name "*.yml" -exec bash -c 'iconv -f ISO-8859-1 -t UTF-8 "{}" -o "{}.utf8" && mv "{}.utf8" "{}"' \;
```

### 2. Erreur : `Missing sudo password`
### Erreur :
```sh
fatal: [srv-pki.itway.local]: FAILED! => {"msg": "Missing sudo password"}
```

>[!Causes]
>`become: true` était activé, mais l’utilisateur `ansible` n'était **pas autorisé à utiliser sudo sans mot de passe**
### Solution :
On édite la configuration sudo via `visudo` sur la machine distante :
```bash
visudo
```

Ajouter la ligne suivante à la fin du fichier :
```bash
ansible ALL=(ALL) NOPASSWD: ALL
```

### 3. Erreur : `sudo must be owned by uid 0 and have the setuid bit set`
### Erreur :
```sh
sudo: /usr/bin/sudo must be owned by uid 0 and have the setuid bit set
```

>[!Causes]
>- Les permissions sur `/usr/bin/sudo` ont été altérées
>- Le binaire n’est plus exécutable avec élévation
### Solution :
Réparation des droits sudo :
```bash
chown root:root /usr/bin/sudo
chmod 4755 /usr/bin/sudo
```

Vérification :
```bash
ls -l /usr/bin/sudo
```

