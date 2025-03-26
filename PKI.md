# SERVEUR PKI - Déploiement Automatisé avec Ansible

## Organisation des rôles et fichiers Ansible

```
├── ansible.cfg
├── inventory/
│   └── hosts
├── playbooks/
│   └── pki.yml
└── roles/
    └── pki/
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

## 5. Tâches du rôle PKI (version améliorée)

```yaml
# roles/pki/tasks/main.yml
- name: Installer les paquets nécessaires
  apt:
    name:
      - openssl
      - ca-certificates
    state: present
    update_cache: yes
  tags: installation

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

... (le reste suit dans le fichier complet)
```

# (Suite du contenu dans le fichier Markdown généré)
