<!-- TOC -->
* [Déployer un script de backup et de restauration pour une instance OpenSILEX](#déployer-un-script-de-backup-et-de-restauration-pour-une-instance-opensilex)
  * [Présentation du script de sauvegarde](#présentation-du-script-de-sauvegarde)
  * [Présentation du script de restauration](#présentation-du-script-de-restauration)
  * [Utilisation du playbook pour effectuer le déploiement](#utilisation-du-playbook-pour-effectuer-le-déploiement)
    * [Variables de connexion à l'hôte](#variables-de-connexion-à-lhôte)
    * [Variables de paramétrage du déploiement](#variables-de-paramétrage-du-déploiement)
    * [Variables de configuration des scripts](#variables-de-configuration-des-scripts)
  * [Recommandations et liens utiles](#recommandations-et-liens-utiles)
    * [Utiliser des groupes d'hôtes pour mutualiser des variables](#utiliser-des-groupes-dhôtes-pour-mutualiser-des-variables)
    * [Protéger les secrets avec Ansible Vault](#protéger-les-secrets-avec-ansible-vault)
  * [Lignes de commandes](#lignes-de-commandes)
  * [Exemple de fichier d'inventaire](#exemple-de-fichier-dinventaire)
<!-- TOC -->

# Déployer un script de backup et de restauration pour une instance OpenSILEX

Le playbook Ansible `playbook_deploy_backup.yml` permet de déployer, sur une machine où une instance OpenSILEX est
installée, un script de sauvegarde (backup) et un script de restauration d'anciennes backup.

Le déploiement consiste à construire ces scripts à partir des variables définies dans l'inventaire, les envoyer sur
la machine hôte, mettre en place le crontab et la rotation des logs ainsi que la configuration SSH pour accéder au
server de stockage distant.

## Présentation du script de sauvegarde

Le script de sauvegarde est construit à partir du template `opensilex-backup.bash.j2`. Une fois configuré correctement,
il effectue les opérations suivantes :

- Créer un dossier de sauvegarde contenant les éléments suivants s'ils sont paramétrés correctement :
    - Une archive des fichiers binaires et de configuration de l'instance OpenSILEX
    - Une archive des logs de l'instance
    - Une archive des fichiers stockés sur le système de fichier local
    - Un dump MongoDB de la base utilisée par l'instance
    - Un export du dépôt RDF utilisé par l'instance
- Envoyer la backup sur le système de stockage distant (s'il est paramétré)
- Supprimer les anciennes backups en local (dont la date est supérieure à la durée d'expiration paramétrée)
- Supprimer les anciennes backups sur le système de stockage distant (en conservant une backup par mois)

## Présentation du script de restauration

Le script de restauration est construit à partir du template `opensilex-restore.bash.j2`. Il s'agit d'un script
interactif permettant de restorer une sauvegarde à partir d'un dossier construit par le script de déploiement. Il
agit de la manière suivante :

- Recherche les sauvegardes disponibles dans le dossier configuré, et demande à l'utilisateur d'en choisir une (par
  défaut, il prendra la plus récente)
- Pour chacun de ces éléments, vérifie qu'il est présent et propose à l'utilisateur de remplacer l'état actuel de
  l'instance par l'élément en question :
    - L'archive des fichiers binaires
    - L'archive des logs de l'instance
    - L'archive des fichiers stockés sur le système de fichier local
    - Le dump MongoDB de la base utilisée par l'instance
    - L'export du dépôt RDF utilisé par l'instance

## Utilisation du playbook pour effectuer le déploiement

Pour utiliser le playbook, il est nécessaire de définir un certain nombre de variables dans l'inventaire.

### Variables de connexion à l'hôte

Pour configurer la connexion aux hôtes, le playbook ne définit pas de méthode particulière. Le comportement par défaut
d'Ansible est d'initier une connexion SSH en se basant sur la configuration du système où il s'exécute, à partir du
nom de l'hôte définit dans l'inventaire.

Pour surcharger ce comportement, par exemple définir un autre utilisateur que celui défini dans la configuration SSH,
vous pouvez définir les **variables de connexion**.

Voici quelques exemples de variables usuelles :

| Variable                       | Description                                                                                                                                                 |
|--------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `ansible_user`                 | L'utilisateur avec lequel Ansible initie la connexion SSH à l'hôte. L'utilisateur doit avoir les droits d'écriture sur les dossiers ciblés par le playbook. |
| `ansible_host`                 | L'adresse IP ou le nom d'hôte ciblé par la connexion SSH.                                                                                                   |
| `ansible_ssh_private_key_file` | Le chemin vers la clé privée utilisée pour la connexion SSH.                                                                                                |

Pour plus d'informations sur les variables de connexion, référez-vous à
[la documentation d'Ansible](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html#connection-variables)
sur le sujet.

### Variables de paramétrage du déploiement

Ces variables servent à paramétrer l'emplacement des scripts sur la machine hôte, des logs générés, et de la rotation
des logs. Elles permettent aussi de définir le déploiement d'une clé SSH pour se connecter au serveur de stockage
distant si besoin.

Les variables marquées par un astérisque sont obligatoires.

| Variable                    | Description                                                                                                                                                                                                                                                                                                                                                                                                                         |
|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `backup_deploy_location`*   | Le dossier sur la machine hôte où seront copiés les scripts. L'utilisateur doit avoir les droits d'écriture.                                                                                                                                                                                                                                                                                                                        |
| `backup_log_location`*      | Le dossier sur la machine hôte où les logs des scripts seront générés. L'utilisateur doit avoir les droits d'écriture sur l'emplacement.                                                                                                                                                                                                                                                                                            |
| `backup_logrotate_location` | Le dossier sur la machine hôte où seront copiés les fichiers de configuration et de statut de `logrotate`. Laisser vide pour ne pas activer la rotation des logs de sauvegarde, où si la rotation est déjà gérée (par exemple dans le dossier `/var/log`)                                                                                                                                                                           |
| `remote_ssh_key`            | Le contenu de la clé privée permettant de se connecter au serveur de stockage distant, si le script est paramétré avec cette option. Le contenu sera copié dans le fichier défini par `j2_remote_backup_ssh_keyfile`. Attention, il est recommandé de ne pas stocker en clair la clé privée. Privilégier l'utilisation d'un système sécurisé comme [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html). |
| `remote_ssh_key_public`     | Le contenu de la clé publique correspondant à la clé privée `remote_ssh_key`.                                                                                                                                                                                                                                                                                                                                                       |


### Variables de configuration des scripts

Ces variables correspondent aux variables de configuration des scripts de déploiement et de restauration. Elles
définissent l'emplacement où les sauvegardes seront placées, les différents éléments de l'instance, et la configuration
du serveur de stockage distant.

Seule l'emplacement des sauvegardes est obligatoire.

| Variable                       | Description                                                                                                                                                                                                                                                                                        |
|--------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `j2_backup_directory`*         | Le dossier sur la machine hôte où seront stockées les sauvegardes. L'utilisateur doit avoir les droits en écriture.                                                                                                                                                                                |
| `j2_bin_directory`             | Le dossier contenant les exécutables et le fichier de configuration de l'instance OpenSILEX. Laisser vide pour ne pas les sauvegarder.                                                                                                                                                             |
| `j2_logs_directory`            | Le dossier contenant les logs de l'instance. Laisser vide pour ne pas sauvegarder les logs.                                                                                                                                                                                                        |
| `j2_data_directory`            | Le dossier contenant les fichiers de l'instance stockés sur le système de fichier. Laisser vide pour ne pas les sauvegarder.                                                                                                                                                                       |
| `j2_mongo_host`                | L'hôte MongoDB auquel l'instance est connectée. Par exemple, `localhost:27017`. Laisser vide pour ne pas sauvegarder la base MongoDB. Nécessite que `j2_mongo_database` soit défini.                                                                                                               |
| `j2_mongo_database`            | La base MongoDB à laquelle l'instance est connectée. N'a d'effet que si `j2_mongo_host` est défini.                                                                                                                                                                                                |
| `j2_rdf4j_host`                | Le serveur RDF4J/GraphDB auquel l'instance est connectée. Par exemple, `http://localhost:8080/rdf4j-server` ou `http://localhost:7200`. Laisser vide pour ne pas sauvegarder le dépôt RDF. Nécessite que `j2_rdf4j_repository` soit défini.                                                        |
| `j2_rdf4j_repository`          | Le dépôt RDF auquel l'instance est connectée. N'a d'effet que si `j2_rdf4j_host` est défini.                                                                                                                                                                                                       |
| `j2_backup_retaining_duration` | La durée de rétention des sauvegardes. Doit être dans un format accepté par la commande `date --date`. Par exemple, `-30 day` permet de conserver les backups 30 jours. Laisser vide pour ne pas supprimer les anciennes backups automatiquement.                                                  |
| `j2_remote_backup_login`       | L'utilisateur sur serveur distant que le script utilisera pour la connection SSH.                                                                                                                                                                                                                  |
| `j2_remote_backup_host`        | Le serveur distant où le script enverra les sauvegardes par `rsync`. Laisser vide pour ne pas activer la réplication des sauvegardes sur serveur distant. Nécessite que les autres variables `j2_remote_*` soient définies.                                                                        |
| `j2_remote_backup_directory`   | Le dossier du serveur distant où seront copiées les backups.                                                                                                                                                                                                                                       |
| `j2_remote_backup_ssh_keyfile` | Le chemin de la clé privée SSH utilisée par le script pour se connecter au serveur distant. Si `remote_ssh_key` est défini, son contenu est copié dans ce fichier. De même, si `remote_ssh_key_public` est défini, un fichier de clé publique est créé à cet emplacement (avec l'extension `.pub`) |

## Recommandations et liens utiles

### Utiliser des groupes d'hôtes pour mutualiser des variables

Dans la définition d'un inventaire Ansible, le système de groupes d'hôtes est un outil qui permet de mettre en commun
certaines variables et d'éviter de les redéfinir. Toutes les variables définies au niveau d'un groupe sont appliquées
à l'ensemble des hôtes de ce groupe, sauf celles qui sont définies pour un hôte en particulier.

Plus d'informations à ce sujet sur la documentation d'Ansible : [How to build your inventory](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html).

### Protéger les secrets avec Ansible Vault

Les mots de passe ou clé privées SSH ne doivent pas être définies en clair dans des variables. Ansible possède un
système pour encrypter des secrets et y faire référence dans des variables, Ansible Vault.

Lien vers la documentation d'Ansible : [Protecting sensitive data with Ansible vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html).

## Lignes de commandes

Voici quelques lignes de commandes usuelles.

Lien vers la documentation : [ansible-playbook](https://docs.ansible.com/ansible/latest/cli/ansible-playbook.html).

```shell
# Valider le playbook avec un inventaire spécifique (ici "inventory.yml")
ansible-playbook playbook_deploy_backup.yml -i inventory.yml --check

# Lancer le playbook sur cet inventaire
ansible-playbook playbook_deploy_backup.yml -i inventory.yml

# Lancer le playbook sur un group ou un hôte particulier (ici le groupe "phis")
ansible-playbook playbook_deploy_backup.yml -i inventory.yml -l phis

# Demander le mot de passe du vault
ansible-playbook playbook_deploy_backup.yml -i inventory.yml --ask-vault-pass

# Aller chercher le mot de passe du vault dans un fichier (ici "vault-ansible-file")
ansible-playbook playbook_deploy_backup.yml -i inventory.yml --vault-password-file vault-ansible-file
```

## Exemple de fichier d'inventaire

Voici un exemple de fichier d'inventaire pour quelques instances PHIS.

```yaml
all:
  children:
    phis:
      vars:
        # Variables de connexion
        ansible_user: opensilex
        # Paramétrage du déploiement
        backup_deploy_location: /home/opensilex/opensilex-backup
        backup_log_location: "{{ backup_deploy_location }}/log"
        backup_logrotate_location: "{{ backup_log_location }}/logrotate"
        remote_ssh_key: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          [...]
        remote_ssh_key_public: "ssh-ed25519 [...]"
        # Configuration des scripts
        j2_backup_directory: "{{ backup_deploy_location }}/backups"
        j2_bin_directory: "/home/opensilex/opensilex-instances/bin/phis/{{ inventory_hostname }}"
        j2_logs_directory: /home/opensilex/opensilex-instances/logs
        j2_mongo_host: localhost
        j2_mongo_database: "{{ inventory_hostname }}"
        j2_rdf4j_host: http://localhost:8080/rdf4j-server
        j2_rdf4j_repository: "{{ inventory_hostname }}"
        j2_backup_retaining_duration: "-30 day"
        j2_remote_backup_login: negrev
        j2_remote_backup_host: muse-login.meso.umontpellier.fr
        j2_remote_backup_ssh_keyfile: /home/opensilex/backup/remote_ssh_key
        j2_remote_backup_directory: "/home/negrev/backup/{{ inventory_hostname }}"
      hosts:
        m3p: # Configuration identique au groupe, pas de variable à définir
        diaphen:
          j2_mongo_host: 10.0.0.135 # Surcharge d'une variable
        pheno3c:
          j2_mongo_host: 10.0.0.135
        phenotoul:
          j2_mongo_host: 10.0.0.135
        phenotic:
          j2_mongo_host: 10.0.0.135
```

On définit un groupe "phis" contenant les
hôtes "m3p", "diaphen", "pheno3c", "phenotoul" et "phenotic".

Parmi les variables de connexion, on ne définit que `ansible_user`. Ansible utilisera donc la commande
`ssh opensilex@m3p` par exemple pour se connecter à m3p. Cela implique que le nom d'hôte "m3p" doit être connu de SSH,
en le définissant par exemple dans le fichier `.ssh/config`.

Dans les variables de déploiement, on définit d'abord `backup_deploy_location`, qui sera le dossier dans lequel on
déploiera tout ce qui concerne le script de sauvegarde. Comme c'est un dossier à l'intérieur du `home` de
notre utilisateur, on s'assure qu'il a les droits en écriture.

Les emplacements `backup_log_location` et `backup_logrotate_location` sont définis comme des sous-dossiers de
`backup_deploy_location` pour s'assurer des droits de l'utilisateur.

La clé privée `remote_ssh_key` est chiffrée grâce à Ansible Vault, il faudra donc passer le mot de passer à la commande
`ansible-playbook` lors de l'exécution. En revanche, la clé publique est stockée en clair.

Comme toutes les instances PHIS sont architecturées de façon similaire, on peut définir toutes les variables de
configuration des scripts au niveau du groupe. Pour faire référence au nom de l'hôte, on utilise la variable d'Ansible
`inventory_hostname`. Par exemple, `j2_mongo_database` prendra la valeur `m3p` quand le playbook traitera l'hôte "m3p"
et `diaphen` au moment de traiter l'hôte "diaphen".

La valeur définie au niveau du groupe pour `j2_mongo_host` est `localhost`, cependant certaines instances de PHIS se
connectent à un hôte MongoDB distant. On peut donc, pour ces instances, surcharger la valeur de cette variable pour
préciser la bonne adresse.