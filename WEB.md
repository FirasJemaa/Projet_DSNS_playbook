# Configuration de DMZ-WEB
## Objectifs
- Héberger un site (Wordpress par exemple) ou d’autres applis web.
- Configurer SSL/TLS.
- Mettre à jour et durcir régulièrement (Wordpress, plugins, PHP, etc.).
- Appliquer les bonnes pratiques de sécurité en utilisant un utilisateur non-root (`ansible`)
## Prérequis
1. [[ANSIBLE]] **installé** et fonctionnel depuis une machine de gestion (IT-ANSIBLE).
2. [[PKI]] **interne opérationnelle** pour signer les CSR (Certificate Signing Request).

---
## 1. Configuration de la PKI pour le certificat du serveur DMZ-WEB

L’idée est de sécuriser l’accès HTTPS avec un certificat émis par notre propre autorité de certification. Le processus inclut :
1. **Générer une clé privée** et une **CSR** (Certificate Signing Request) sur la machine depuis Ansible.
2. **Transmettre** cette CSR à l’autorité de certification (SRV-PKI).
3. **Signer** la CSR pour obtenir un **certificat**.
4. **Récupérer** ce certificat signé + la chaîne CA.
5. **Installer** la clé privée et le certificat sur la machine `dmz-web`.
6. **Configurer Nginx** pour utiliser ce certificat.

---
## 2. Définition des variables dans `/vars/` et `/group_vars/`

Les variables sont structurées de manière claire :
#### 2.1 `vars/main.yml` :

```yaml
########################################
# roles/dmz-web/vars/main.yml
########################################

# Définit les ports et le dossier de conf Nginx utilisé
dmz_web_port_http: 80
dmz_web_port_https: 443
dmz_web_nginx_conf_dir: "/etc/nginx/sites-available"
```

#### 2.2 `group_vars/dmz-web.yml` :

On définit ici les variables spécifiques à l’hôte `[dmz-web]` :

```yaml
########################################
# inventory/group_vars/dmz-web.yml
########################################

# Définit le FQDN, les chemins de certificat, les DN pour la CSR, et la config PKI
# Ces variables seront injectées automatiquement par Ansible pour ce groupe
dmz_web_fqdn: "dmz-web.itway.fr"
dmz_web_ssl_dir: "/etc/nginx/ssl"
dmz_web_key_filename: "dmz-web.key"
dmz_web_csr_filename: "dmz-web.csr"
dmz_web_crt_filename: "dmz-web.crt"
dmz_web_ca_filename: "ca.crt"

# DN (Distinguished Name) pour la CSR
dmz_web_country: "FR"
dmz_web_state: "Ile-de-France"
dmz_web_org: "ITWay"
dmz_web_ou: "DMZ"

# PKI side
srv_pki_host: "srv-pki.itway.local"     # nom d'hôte dans hosts.ini
pki_openssl_conf: "/etc/ssl/openssl.cnf"
pki_dir: "/etc/pki"

# Paramètres SSL sur Nginx
dmz_web_ssl_protocols: "TLSv1.2 TLSv1.3"
dmz_web_ssl_ciphers: "HIGH:!aNULL:!MD5:!ADH:!SHA1:!DES:!RC4"
dmz_web_root: "/var/www/html"
dmz_web_index: "index.html"
```

>[!Explication]
>Pourquoi utiliser `group_vars` ? Cela permet d'appliquer une config différente selon la machine ciblée tout en gardant les rôles génériques.

---
## 3. Rôle – `tasks/main.yml`

Ce fichier est le point d'entrée du rôle. Il exécute dans l'ordre :
1. Installation de Nginx
2. Import du cycle de certificat (via `cert.yml`)
3. Déploiement des fichiers de configuration
4. Vérification et redémarrage conditionnel de Nginx
5. Déploiement d’un `index.html` pour test
6. Healthcheck automatique HTTPS

#### `tasks/main.yml` :

```yaml
########################################
# roles/dmz-web/tasks/main.yml
########################################

- name: Installer Nginx
  apt:
    name: nginx
    state: present
    update_cache: yes

# Appel des tâches liées à la génération et signature du certificat
- import_tasks: cert.yml

# Déploiement de la configuration Nginx
- name: Déployer le template principal nginx.conf
  template:
    src: "nginx.conf.j2"
    dest: "/etc/nginx/nginx.conf"
  notify: reload nginx

- name: Déployer la configuration du site par défaut (HTTPS)
  template:
    src: "default.conf.j2"
    dest: "/etc/nginx/sites-available/default"
  notify: reload nginx

- name: Déployer les paramètres SSL (ssl_params.conf)
  template:
    src: "ssl_params.conf.j2"
    dest: "/etc/nginx/snippets/ssl_params.conf"
  notify: reload nginx

# Vérifie que le certificat SSL est bien présent avant de tester nginx
- name: Vérifier si le certificat SSL est bien présent
  stat:
    path: "{{ dmz_web_ssl_dir }}/{{ dmz_web_crt_filename }}"
  register: crt_file_check

# Vérifie la validité de la configuration nginx
- name: Vérifier que la configuration Nginx est correcte
  command: nginx -t
  register: nginx_config_test
  failed_when: "'test failed' in nginx_config_test.stderr or nginx_config_test.rc != 0"
  changed_when: false
  when: crt_file_check.stat.exists

# Démarre ou recharge nginx selon son état, via un handler Ansible
- name: Forcer le rechargement de Nginx si la conf est OK
  meta: flush_handlers
  when:
    - crt_file_check.stat.exists
    - "'test is successful' in nginx_config_test.stdout or 'syntax is ok' in nginx_config_test.stdout"

# Assure que le répertoire web existe et appartient à ansible
- name: Créer le répertoire web root s’il n’existe pas
  file:
    path: "{{ dmz_web_root }}"
    state: directory
    owner: www-data
    group: www-data
    mode: "0755"

# Déploie une page index.html simple
- name: Déployer une page index.html de test
  copy:
    dest: "{{ dmz_web_root }}/index.html"
    content: "<h1>Serveur {{ dmz_web_fqdn }} opérationnel </h1>"
    owner: www-data
    group: www-data
    mode: "0644"

# Vérifie que la page HTTPS répond correctement (healthcheck)
- name: Vérifier la disponibilité HTTPS via IP (avec Host header)
  uri:
    url: "https://{{ ansible_host }}"
    headers:
      Host: "{{ dmz_web_fqdn }}"
    validate_certs: no
    return_content: no
    status_code: 200
  register: https_health
  ignore_errors: true

# healthcheck à ajouté ici !

```

### Healthcheck HTTPS
Grâce à la tâche `uri`, Ansible simule un appel HTTPS sur l’IP de la machine mais avec l’en-tête `Host` du FQDN :

```yaml
# A la suite du fichier yaml tasks -> main.yml
- name: Afficher le résultat du healthcheck
  debug:
    msg: >-
      ✔️ Accès HTTPS opérationnel sur {{ dmz_web_fqdn }}
      (Status: {{ https_health.status | default('inconnu') }})
  when: https_health.status is defined and https_health.status == 200

- name: Alerte si le serveur HTTPS ne répond pas
  debug:
    msg: "❌ Problème d’accès HTTPS sur {{ dmz_web_fqdn }} — Code : {{ https_health.status | default('aucun') }}"
  when: https_health.status is not defined or https_health.status != 200
```

Cela permet de s’assurer que :
- Nginx est bien en écoute sur 443
- Le certificat est bien chargé
- Le nom d’hôte correspond

---
## 4. `cert.yml` – Génération, signature, récupération du certificat

Ce fichier encapsule tout le cycle de vie d’un certificat :
- Génération d’une clé privée et CSR côté `dmz-web`
- Fetch de la CSR sur le contrôleur Ansible
- Signature déléguée sur `srv-pki` avec `openssl ca`
- Récupération du certificat signé (`.crt`) + `ca.crt`
- Copie de ces fichiers sur la cible finale

#### `/tasks/cert.yml`:

```yaml
########################################
# roles/dmz-web/tasks/cert.yml
########################################

- block:
    # 1) Générer clé et CSR sur DMZ-WEB
    - name: "[CERT] Créer le répertoire SSL"
      file:
        path: "{{ dmz_web_ssl_dir }}"
        state: directory
        mode: '0755'

    - name: "[CERT] Générer la clé privée"
      command: >
        openssl genrsa -out {{ dmz_web_ssl_dir }}/{{ dmz_web_key_filename }} 4096
      args:
        creates: "{{ dmz_web_ssl_dir }}/{{ dmz_web_key_filename }}"

    - name: "[CERT] Générer la CSR"
      command: >
        openssl req -new
        -key {{ dmz_web_ssl_dir }}/{{ dmz_web_key_filename }}
        -out {{ dmz_web_ssl_dir }}/{{ dmz_web_csr_filename }}
        -subj "/C={{ dmz_web_country }}/ST={{ dmz_web_state }}/O={{ dmz_web_org }}/OU={{ dmz_web_ou }}/CN={{ dmz_web_fqdn }}"
      args:
        creates: "{{ dmz_web_ssl_dir }}/{{ dmz_web_csr_filename }}"

    # 2) Créer le répertoire local de stockage sur le contrôleur
    - name: "[LOCAL] Créer le dossier files/dmz-web sur le contrôleur"
      local_action:
        module: file
        path: "files/dmz-web"
        state: directory
        mode: "0755"

    # 3) Récupérer la CSR vers le contrôleur
    - name: "[CERT] fetch la CSR -> Ansible controller"
      fetch:
        src: "{{ dmz_web_ssl_dir }}/{{ dmz_web_csr_filename }}"
        dest: "files/dmz-web/{{ dmz_web_csr_filename }}"
        flat: yes

    # 4) Initialisation PKI côté srv-pki
    - name: "[PKI] Vérifier si le fichier serial existe"
      stat:
        path: "{{ pki_dir }}/serial"
      register: serial_file
      delegate_to: "{{ srv_pki_host }}"
      become: yes

    - name: "[PKI] Initialiser serial à 01 si absent ou vide"
      copy:
        dest: "{{ pki_dir }}/serial"
        content: "01\n"
        mode: "0644"
      when: not (serial_file.stat.exists and serial_file.stat.size > 0)
      delegate_to: "{{ srv_pki_host }}"
      become: yes

    - name: "[PKI] Vérifier si index.txt existe"
      stat:
        path: "{{ pki_dir }}/index.txt"
      register: index_file
      delegate_to: "{{ srv_pki_host }}"
      become: yes

    - name: "[PKI] Créer index.txt vide si absent"
      copy:
        dest: "{{ pki_dir }}/index.txt"
        content: ""
        mode: "0644"
      when: not index_file.stat.exists
      delegate_to: "{{ srv_pki_host }}"
      become: yes

    - name: "[PKI] Vérifier si crlnumber existe"
      stat:
        path: "{{ pki_dir }}/crlnumber"
      register: crl_file
      delegate_to: "{{ srv_pki_host }}"
      become: yes

    - name: "[PKI] Initialiser crlnumber à 01 si absent ou vide"
      copy:
        dest: "{{ pki_dir }}/crlnumber"
        content: "01\n"
        mode: "0644"
      when: not (crl_file.stat.exists and crl_file.stat.size > 0)
      delegate_to: "{{ srv_pki_host }}"
      become: yes

    - name: "[PKI] Créer le dossier reqs/ sur SRV-PKI"
      file:
        path: "{{ pki_dir }}/reqs"
        state: directory
        mode: "0755"
      delegate_to: "{{ srv_pki_host }}"
      become: yes

    # 5) Copier la CSR depuis Ansible -> srv-pki
    - name: "[CERT] Copier la CSR de Ansible -> SRV-PKI"
      copy:
        src: "files/dmz-web/{{ dmz_web_csr_filename }}"
        dest: "{{ pki_dir }}/reqs/{{ dmz_web_csr_filename }}"
        mode: '0644'
      delegate_to: "{{ srv_pki_host }}"
      become: yes

    # 6) Signer la CSR
    - name: "[CERT] Signer la CSR sur SRV-PKI"
      command: >
        openssl ca
        -config {{ pki_openssl_conf }}
        -extensions v3_server
        -in {{ pki_dir }}/reqs/{{ dmz_web_csr_filename }}
        -out {{ pki_dir }}/certs/{{ dmz_web_crt_filename }}
        -batch
      args:
        creates: "{{ pki_dir }}/certs/{{ dmz_web_crt_filename }}"
      delegate_to: "{{ srv_pki_host }}"
      become: yes

    - name: "[PKI] Vérifier si le CRT généré existe"
      stat:
        path: "{{ pki_dir }}/certs/{{ dmz_web_crt_filename }}"
      register: crt_stat
      delegate_to: "{{ srv_pki_host }}"
      become: yes

    - name: "[CERT] Vérifier le certificat généré"
      command: "openssl x509 -in {{ pki_dir }}/certs/{{ dmz_web_crt_filename }} -noout -text"
      register: crt_info
      changed_when: false
      when: crt_stat.stat.exists
      delegate_to: "{{ srv_pki_host }}"
      become: yes

    - name: "DEBUG - Date d'expiration du CRT"
      debug:
        msg: "{{ crt_info.stdout | regex_search('Not After : (.+)') }}"
      delegate_to: localhost
      changed_when: false
      when: crt_stat.stat.exists

    # 7) Récupérer le CRT depuis srv-pki
    - name: "[CERT] fetch le CRT -> Ansible"
      fetch:
        src: "{{ pki_dir }}/certs/{{ dmz_web_crt_filename }}"
        dest: "files/dmz-web/{{ dmz_web_crt_filename }}"
        flat: yes
      when: crt_stat.stat.exists
      delegate_to: "{{ srv_pki_host }}"
      become: yes

    - name: "[CERT] fetch la CA -> Ansible"
      fetch:
        src: "{{ pki_dir }}/certs/ca.crt"
        dest: "files/dmz-web/ca.crt"
        flat: yes
      delegate_to: "{{ srv_pki_host }}"
      become: yes

    # 8) Déploiement sur DMZ-WEB
    - name: "[CERT] Copier le CRT signé sur DMZ-WEB"
      copy:
        src: "files/dmz-web/{{ dmz_web_crt_filename }}"
        dest: "{{ dmz_web_ssl_dir }}/{{ dmz_web_crt_filename }}"
        mode: '0644'

    - name: "[CERT] Copier la CA sur DMZ-WEB"
      copy:
        src: "files/dmz-web/ca.crt"
        dest: "{{ dmz_web_ssl_dir }}/{{ dmz_web_ca_filename }}"
        mode: '0644'

  rescue:
    - name: "Erreur dans la génération/signature du certificat"
      debug:
        msg: "Erreur attrapée : problème lors de la signature du certificat sur la PKI"

```

>[!Explication]
>**Explications** :
>1. **Clé + CSR** générées localement sur DMZ-WEB (pas d’upload de clé privée, c’est plus sûr).
>2. **fetch** → la CSR arrive sur Ansible.
>3. **delegate_to : srv_pki_host** pour signer la CSR sur SRV-PKI.
>4. **fetch** → on rapatrie le `.crt` signé (et la `ca.crt`) vers Ansible.
>5. **copy** → On envoie `.crt` et `ca.crt` sur DMZ-WEB.

Aucune connexion directe PKI ↔ DMZ n’est nécessaire. Tout passe par Ansible.

---
## 5. Fichier `roles/dmz-web/handlers/main.yml`

Le **handler** est utilisé pour redémarrer ou recharger Nginx de manière conditionnelle lorsque des fichiers de configuration sont modifiés. Il est déclenché via la directive `notify` : 

```yaml
########################################
# roles/dmz-web/handlers/main.yml
########################################
- name: reload nginx
  service:
    name: nginx
    state: reloaded
```

Les **handlers** s’exécutent à la fin, si quelque chose a changé.

---
## 6. Templates Nginx
### 6.1 Fichier `roles/dmz-web/templates/nginx.conf.j2`

Le fichier **`nginx.conf.j2`** est un **template Jinja2** utilisé par Ansible pour **générer le fichier de configuration principal de Nginx** (`/etc/nginx/nginx.conf`) sur la machine cible.

Il définit la **configuration globale de Nginx**, notamment :

| Élément                               | Rôle                                                                          |
| ------------------------------------- | ----------------------------------------------------------------------------- |
| `user www-data;`                      | Spécifie l’utilisateur système qui exécutera les processus Nginx              |
| `worker_processes auto;`              | Définit le nombre de processus Nginx à lancer en parallèle (auto = selon CPU) |
| `pid /run/nginx.pid;`                 | Emplacement du fichier PID (process ID)                                       |
| `events { worker_connections 768; }`  | Nombre max de connexions simultanées par processus                            |
| `http { ... }`                        | Bloc principal pour tous les serveurs HTTP (vhosts)                           |
| `include /etc/nginx/sites-enabled/*;` | Charge les vhosts comme celui défini dans `default.conf.j2`                   |
#### `roles/dmz-web/templates/nginx.conf.j2` :

```jinja2
# Configuration principale de Nginx, utilisée globalement
user  www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
  worker_connections 768;
}

http {
  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  sendfile        on;
  keepalive_timeout 65;

  # Journaux
  access_log /var/log/nginx/access.log;
  error_log  /var/log/nginx/error.log;

  include /etc/nginx/conf.d/*.conf;
  include /etc/nginx/sites-enabled/*;
}
```

>[!Pourquoi c'est important ?]
>- Tu **maîtrises complètement la conf globale** de Nginx (aucun résidu de l’installation par défaut)
>- Tu peux y intégrer des **politiques de sécurité** : logs, timeout, etc.
>- Tu centralises la gestion via Ansible → auditabilité + reproductibilité

### 6.2 Fichier `roles/dmz-web/templates/default.conf.j2`

Le fichier **`roles/dmz-web/templates/default.conf.j2`** est un **template Jinja2** qui sert à générer **la configuration d’un site web (vhost)** dans Nginx.

Il définit le **vhost** principal pour le serveur, c’est-à-dire :
- Quel nom d’hôte Nginx doit servir (`server_name`)
- Sur quel port écouter (`listen 80`, `listen 443 ssl`)
- Où sont les fichiers web (`root`, `index`)
- Et surtout : comment gérer le **HTTPS avec certificat signé par ta PKI**

```jinja2
# Premier bloc serveur : écoute sur HTTP (port 80) uniquement pour rediriger vers HTTPS
server {
    listen 80 default_server;               # Écoute en IPv4 sur le port 80, défini comme site par défaut
    listen [::]:80 default_server;          # Écoute en IPv6 également

    server_name {{ dmz_web_fqdn }};         # Nom d’hôte attendu (ex: dmz-web.itway.fr)

    # Redirection automatique de toutes les requêtes HTTP vers HTTPS
    return 301 https://$host$request_uri;   # Redirige de manière permanente vers l’équivalent HTTPS
}
# Ce bloc permet de **forcer le passage en HTTPS** dès qu’un utilisateur accède au site via HTTP.

# Bloc principal HTTPS
server {
    listen 443 ssl http2 default_server;       # Écoute sur le port 443 avec SSL/TLS activé
    listen [::]:443 ssl http2 default_server;  # Idem pour IPv6

    server_name {{ dmz_web_fqdn }};            # Nom d’hôte pour lequel ce vhost s'applique

    # Chemins vers les certificats TLS
    ssl_certificate     {{ dmz_web_ssl_dir }}/{{ dmz_web_crt_filename }};         # Certificat signé
    ssl_certificate_key {{ dmz_web_ssl_dir }}/{{ dmz_web_key_filename }};         # Clé privée associée
    ssl_trusted_certificate {{ dmz_web_ssl_dir }}/{{ dmz_web_ca_filename }};      # Certificat de la CA (chaîne complète)

    # Chargement de paramètres SSL durcis (ciphers, protocoles, etc.)
    include /etc/nginx/snippets/ssl_params.conf;

    # Répertoire racine du site web (ex: /var/www/html)
    root {{ dmz_web_root }};
    index {{ dmz_web_index }};                 # Fichier à servir par défaut (ex: index.html)

    # Bloc de gestion des requêtes sur la racine /
    location / {
        try_files $uri $uri/ =404;             # Essaie d’afficher le fichier demandé, sinon renvoie une erreur 404
    }
}
```

#### Résumé des pratiques utilisées dans ce fichier :

| Bonne pratique                                | Pourquoi c’est important                           |
| --------------------------------------------- | -------------------------------------------------- |
| Redirection HTTP → HTTPS                      | Oblige à utiliser HTTPS                            |
| Certificat signé par CA                       | Confiance garantie en interne                      |
| Paramètres SSL durcis (via `ssl_params.conf`) | Pour une sécurité maximale                         |
| Support HTTP/2 activé                         | Améliore la performance                            |
| Gestion d’erreurs (try_files)                 | Évite les erreurs de type directory listing ou 500 |
| Vhost par nom (via `server_name`)             | Permet de gérer plusieurs sites si besoin          |

### 6.3 Fichier `roles/dmz-web/templates/ssl_params.conf.j2`

Le fichier **`roles/dmz-web/templates/ssl_params.conf.j2`** est un **template Jinja2** utilisé pour définir une configuration SSL/TLS **renforcée** pour Nginx.  
Il est **inclus** dans ton vhost principal (`default.conf.j2`) via :

```nginx
include /etc/nginx/snippets/ssl_params.conf;
```

Ce fichier contient des **paramètres cryptographiques avancés** pour :

- **Choisir les protocoles TLS autorisés**    
- **Limiter les algorithmes de chiffrement (ciphers)** à ceux considérés comme forts    
- **Optimiser les performances des connexions SSL**    
- **Renforcer la sécurité en forçant les préférences serveur**

```jinja2
ssl_protocols {{ dmz_web_ssl_protocols }};
# Définit les versions de TLS autorisées (ex: TLSv1.2, TLSv1.3).
# En bloquant les versions obsolètes comme SSLv3, TLSv1.0 et TLSv1.1, on évite de nombreuses failles (ex: POODLE).

ssl_ciphers {{ dmz_web_ssl_ciphers }};
# Liste des suites de chiffrement autorisées.
# On exclut les ciphers faibles comme RC4, DES, MD5, etc.
# Exemple de valeur : "HIGH:!aNULL:!MD5:!ADH:!SHA1:!DES:!RC4"

ssl_prefer_server_ciphers on;
# Force le serveur à imposer ses propres préférences de chiffrement plutôt que celles du client.
# Cela garantit qu’on utilise es ciphers les plus sécurisés disponibles.

ssl_session_cache shared:SSL:10m;
# Active la mise en cache des sessions SSL pour améliorer les performances.
# Permet à un client qui se reconnecte de ne pas refaire une négociation complète.
# 10m = cache jusqu’à 10 Mo de sessions SSL partagées.

ssl_session_timeout 10m;
# Définit la durée de vie d'une session SSL mise en cache.
# Après 10 minutes, la session expirera et une nouvelle négociation sera nécessaire.
```

#### Résumé des commandes :
|Ligne|Ce qu’elle fait|
|---|---|
|`ssl_protocols`|Bloque les protocoles obsolètes (ex: SSLv3)|
|`ssl_ciphers`|Choisit des ciphers modernes et forts|
|`ssl_prefer_server_ciphers`|Donne priorité aux ciphers choisis par le serveur|
|`ssl_session_cache`|Optimise les performances SSL|
|`ssl_session_timeout`|Gère la durée des sessions SSL mises en cache

---
## 7. Inventaire Ansible 

Dans `inventory/hosts.ini`, Ajouter la ligne suivante :

```ini
[dmz-web]
10.10.10.2 ansible_user=ansible ansible_ssh_private_key_file=~/.ssh/id_ed25519 ansible_become=true ansible_become_method=sudo
```

---
## 8. Exécution le rôle avec un playbook `setup-web.yml`
Ce **playbook** applique le rôle `dmz-web` sur le groupe `[dmz-web]` :

```yaml
########################################
# playbooks/setup-web.yml
########################################
- name: Déployer le serveur web + Cert SSL complet
  hosts: dmz-web
  become: yes
  roles:
    - dmz-web
```
### Déploiement :

```sh
ansible-playbook -i inventory/hosts.ini playbooks/setup-web.yml
```

---
## 9. Problèmes rencontrés
### Impossible d’utiliser `service nginx start`

Au début, la tâche suivante échouait :
```yaml
- name: Démarrer nginx
  command: service nginx start
```

### Pourquoi ça bloquait :
- La machine `dmz-web` est un conteneur **sans `systemd`** ni support de `start-stop-daemon`
- `service` tentait un `setuid/setgid` refusé par le noyau : erreur `Operation not permitted`

### Analyse :
- Test manuel dans le conteneur avec `nginx -t` et `nginx` : OK
- `pgrep nginx` pour vérifier s’il tourne
- Utilisation d’un handler `reload nginx` conditionné par la validité de la config

### Solution mise en place :
- Remplacement du bloc `service` par :

```yaml
- name: reload nginx
  shell: |
    if pgrep nginx > /dev/null; then
      nginx -s reload
    else
      nginx
    fi
```

Cela fonctionne même dans des conteneurs minimalistes, tout en gardant la logique Ansible (`notify`, `handlers`, `meta: flush_handlers`).

