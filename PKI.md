# SERVEUR PKI - Déploiement Automatisé avec Ansible

## Organisation des rôles et fichiers Ansible

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
srv-pki.itway.local ansible_host=172.16.50.3 ansible_user=ansible ansible_ssh_private_key_file=~/.ssh/id_rsa
```

**Commentaires :**
- L'utilisation d'un FQDN est recommandée pour les PKI
- L’authentification SSH par clé **est déjà en place**, ce qui assure une connexion sécurisée
## 3. Variables pour la PKI `roles/pki/vars/main.yml`

```yaml
# Paramètres de base PKI
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

# Paramètres techniques (ajoutés)
pki_key_bits: 4096
pki_ca_days: 3650
pki_default_md: "sha256"
pki_crl_days: 30

# Permissions (ajoutées)
pki_private_dir_mode: "0700"
pki_key_mode: "0600"
pki_cert_mode: "0644"
```

**Commentaires :**
- Tous les paramètres techniques sont maintenant externalisés en variables
- Ajout de paramètres de sécurité (permissions, algorithmes)
- La durée de validité est configurable (10 ans par défaut pour la CA)

## 4. Configuration OpenSSL 'templates/main.yml`

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
crl_days        = {{ pki_crl_days }}
unique_subject  = no

# Nouvelle section pour les certificats serveur
[ v3_server ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical,CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

# Nouvelle section pour les certificats clients
[ v3_client ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical,CA:FALSE
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

- name: Installer les paquets nécessaires
  apt:
    name:
      - openssl
      - ca-certificates
    state: present
    update_cache: yes
  tags: installation
  notify: restart openssl
  comment: |
    Installation des paquets de base pour la PKI
    - openssl : outils cryptographiques
    - ca-certificates : bundle de certificats racines

- name: Créer l'arborescence PKI
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
  comment: |
    Création de l'arborescence PKI standard
    - /private a des permissions restrictives (0700 par défaut)
    - Autres dossiers en 755

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
  comment: |
    Création des fichiers de suivi OpenSSL
    - index.txt : base de données des certificats
    - serial : numéro de série incrémental
    - crlnumber : suivi des révocation

- name: Initialiser le numéro de série
  copy:
    dest: "{{ pki_ca_dir }}/serial"
    content: "01"
    mode: "0644"
  when: not serial_file.stat.exists
  vars:
    serial_file: "{{ lookup('stat', pki_ca_dir + '/serial') }}"
  tags: setup
  comment: |
    Initialise le fichier serial avec '01' si inexistant
    Conditionné par when: pour éviter l'écrasement

- name: Initialiser le numéro de CRL
  copy:
    dest: "{{ pki_ca_dir }}/crlnumber"
    content: "01"
    mode: "0644"
  when: not crl_file.stat.exists
  vars:
    crl_file: "{{ lookup('stat', pki_ca_dir + '/crlnumber') }}"
  tags: setup
  comment: "Même principe que pour le serial mais pour les CRL"

- name: Créer le fichier index.txt vide
  copy:
    dest: "{{ pki_ca_dir }}/{{ pki_index_file }}"
    content: ""
    mode: "0644"
  when: not index_file.stat.exists
  vars:
    index_file: "{{ lookup('stat', pki_ca_dir + '/' + pki_index_file) }}"
  tags: setup
  comment: "Fichier vide pour la base de données des certificats"

- name: Déployer le fichier de configuration OpenSSL
  template:
    src: openssl.cnf.j2
    dest: "{{ pki_openssl_conf }}"
    owner: root
    group: root
    mode: "0644"
  tags: configuration
  comment: |
    Déploie le template Jinja2 personnalisé
    Utilise les variables définies dans vars/main.yml

- name: Générer le fichier aléatoire OpenSSL
  command: openssl rand -out {{ pki_ca_dir }}/private/.rand 4096
  args:
    creates: "{{ pki_ca_dir }}/private/.rand"
  tags: setup
  comment: "Fichier de graine aléatoire pour la génération de clés"

- name: Sécuriser les permissions des dossiers
  file:
    path: "{{ item }}"
    mode: "0750"
    owner: root
    group: root
  loop:
    - "{{ pki_ca_dir }}"
    - "{{ pki_ca_dir }}/private"
  tags: security
  comment: "Double vérification des permissions"

# ==================== #
# PARTIE CERTIFICATS CA #
# ==================== #

- name: Générer la clé privée de la CA
  command: >
    openssl genrsa
    -out {{ pki_ca_dir }}/private/{{ pki_ca_key_name }}
    {{ pki_key_bits }}
  args:
    creates: "{{ pki_ca_dir }}/private/{{ pki_ca_key_name }}"
  notify: Secure CA private key
  tags: certificates
  comment: |
    Génère la clé RSA 4096 bits par défaut
    - creates: évite la regénération si existe déjà
    - notify: déclenche le handler de sécurisation

- name: Générer le certificat auto-signé de la CA
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
  comment: |
    Crée le certificat auto-signé (root CA)
    - Valide 10 ans par défaut (pki_ca_days)
    - Utilise les extensions v3_ca définies dans le template

- name: Valider le certificat CA
  command: openssl x509 -in {{ pki_ca_dir }}/certs/{{ pki_ca_cert_name }} -noout -text
  register: cert_check
  changed_when: false
  tags: validation
  comment: "Vérification technique du certificat généré"

- name: Afficher les infos du certificat
  debug:
    msg: |
      CA générée avec succès !
      Validité: {{ cert_check.stdout | regex_search('Not After : (.+)') }}
      Sujet: {{ cert_check.stdout | regex_search('Subject: (.+)') }}
  when: cert_check is defined
  tags: validation

# ================== #
# GESTION DES CRL    #
# ================== #

- name: Générer le CRL initial
  command: >
    openssl ca -gencrl
    -config {{ pki_openssl_conf }}
    -out {{ pki_ca_dir }}/crl/crl.pem
  args:
    creates: "{{ pki_ca_dir }}/crl/crl.pem"
  tags: crl
  comment: "Génère la première liste de révocation (vide)"

- name: Vérifier le CRL
  command: openssl crl -in {{ pki_ca_dir }}/crl/crl.pem -noout -text
  register: crl_check
  changed_when: false
  when: crl_check is defined
  tags: validation``
```

## Fichier handler associé ```roles/pki/handlers/main.yml``` :
```
- name: Secure CA private key
  file:
    path: "{{ pki_ca_dir }}/private/{{ pki_ca_key_name }}"
    mode: "{{ pki_key_mode | default('0600') }}"
    owner: root
    group: root
  comment: "Sécurise la clé privée après sa création"

- name: restart openssl
  service:
    name: openssl
    state: restarted
  when: false  # Désactivé car pas de service dédié
  comment: "Exemple pour d'autres services qui utiliseraient OpenSSL"
```
## 6 Playbook principal `playbooks/pki.yml`
```
# playbooks/pki.yml
---
- name: Déploiement complet du serveur PKI
  hosts: srv-pki
  become: yes
  
  pre_tasks:
    - name: Vérifier que le système est compatible
      assert:
        that:
          - ansible_os_family == "Debian"
          - ansible_distribution_major_version | int >= 10
        msg: "Ce playbook nécessite Debian 10 ou supérieur"

  roles:
    - srv-pki

  post_tasks:
    - name: Afficher les informations du certificat
      debug:
        msg: "CA déployée avec succès. Certificat valide jusqu'au {{ cert_check.stdout | regex_search('Not After : (.+)') }}"
      when: cert_check is defined
```

Commentaires :

Ajout de pré-vérifications système

Affichage des informations de validation en fin de déploiement

Meilleure gestion des erreurs

## 7 Points de sécurité et bonnes pratiques 
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

## 8 Exemple de signature de certificat serveur (bonus)
```
- name: Générer une CSR pour un serveur
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
Cette version améliorée inclut :

Toutes les suggestions de sécurité

Une meilleure gestion des permissions

Des validations supplémentaires

Des exemples étendus

Une documentation plus complète

Des bonnes pratiques opérationnelles

Pour le déployer :

```
ansible-playbook -i inventory/hosts playbooks/pki.yml --ask-vault-pass
```
