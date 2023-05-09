<!-- TOC -->
* [Deploy a backup and restore script for an OpenSILEX instance](#deploy-a-backup-and-restore-script-for-an-opensilex-instance)
  * [Deploy script overview](#deploy-script-overview)
  * [Restore script overview](#restore-script-overview)
  * [Using the playbook to deploy the scripts](#using-the-playbook-to-deploy-the-scripts)
    * [Host connexion variables](#host-connexion-variables)
    * [Deployment configuration variables](#deployment-configuration-variables)
    * [Scripts configuration variables](#scripts-configuration-variables)
  * [Recommendations and useful links](#recommendations-and-useful-links)
    * [Use host groups to reuse variables between hosts](#use-host-groups-to-reuse-variables-between-hosts)
    * [Protect you secrets with Ansible Vault](#protect-you-secrets-with-ansible-vault)
  * [Command line](#command-line)
  * [Example inventory file](#example-inventory-file)
<!-- TOC -->

# Deploy a backup and restore script for an OpenSILEX instance

The Ansible playbook `playbook_deploy_backup.yml` allows you to deploy a backup and restore scripts on a host where
OpenSILEX is installed.

The playbook builds these scripts from variables defined in the inventory, sends them to the host machine, set up the
crontab, log rotation, and SSH configuration to push the backups on the remote storage.

## Deploy script overview

The backup script is built from the `opensilex-backup.bash.j2` template file. Once correctly configured, it performs
the following operations :

- Create a backup folder with the following components (if they are correctly configured in the script)
  - An archive of the binary and configuration files of the OpenSILEX instance
  - An archive of the instance's logs
  - An archive of the datafiles stored on the local filesystem
  - A MongoDB dump of the database used by the instance
  - An export of the RDF repository used by the instance
- Send the backup on the remote storage server (if configured)
- Remove the old backups on the host (where the create date is older than the configured expiration date)
- Remove the old backups on the remote storage server (but keeping one backup per month)

## Restore script overview

The restore script is built from the `opensilex-restore.bash.j2` template file. It is an interactive tool that can
restore a backup from a folder built by the backup script. It performs the following operations :

- Look for the available backups on the configured folder, and ask the user to pick one (by default, the most recent
  one will be selected)
- For each of the following items, check that it is present and ask the user if they want to replace the current state
  of the instance with the backup :
  - The archive of binary and configuration files
  - The archive of the logs
  - The archive of the local filesystem datafiles
  - The MongoDB dump
  - The RDF repository export

## Using the playbook to deploy the scripts

In order to use the playbook, you need to define some variables in the inventory.

### Host connexion variables

To configure the host connexion, the playbook does not rely on a specific method. Instead, Ansible manages the connexion
using dedicated variables. By default, it tries to initiate an SSH connexion based on your SSH configuration and the
hostname in the inventory.

You can override this behaviour, for example to connect with a user different from the one in your SSH configuration.
To do that, you need to defined **connexion variables**.

Here are some commonly used connexion variables :

| Variable                       | Description                                                                                                                           |
|--------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| `ansible_user`                 | The user used by Ansible to initiate the SSH connexion. The user must have write permissions on the folders targeted by the playbook. |
| `ansible_host`                 | IP address or hostname of the target machine.                                                                                         |
| `ansible_ssh_private_key_file` | Path to the private key used to initiate the SSH connexion.                                                                           |

For more information, please read 
[the Ansible documentation](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html#connection-variables).

### Deployment configuration variables

These variables configure the location of scripts and logs on the host machine, and the log rotation. You can also
define an SSH key to deploy so that the host can connect to a remote storage server if needed.

Required variables are marked by an asterisk.

| Variable                    | Description                                                                                                                                                                                                                                                                                                                                                                                       |
|-----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `backup_deploy_location`*   | The folder on the host where the script will be sent. The user must have write permissions.                                                                                                                                                                                                                                                                                                       |
| `backup_log_location`*      | The folder on the host where the logs for the backup scripts will be generated. The user must have write permissions.                                                                                                                                                                                                                                                                             |
| `backup_logrotate_location` | The folder on the host where the `logrotate` configuration and status files will be copied. Leave empty to disable log rotation, or if the rotation is already managed (if the logs are generated in `/var/log` for example).                                                                                                                                                                     |
| `remote_ssh_key`            | The content of the private key used to connect to the remote storage server, if the script is configured to do so. The content will be copied in the file specified by `j2_remote_backup_ssh_keyfile`. **WARNING** : you should NOT store the private key directly. You should use a secure system instead, like [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html). |
| `remote_ssh_key_public`     | The content of the public key corresponding to `remote_ssh_key`.                                                                                                                                                                                                                                                                                                                                  |


### Scripts configuration variables

These variables correspond to the configuration variables of the deploy and restore scripts. They defined the backup
location, the different items of the instance to save, and the configuration for the remote storage server.

Only the backup location is required.

| Variable                       | Description                                                                                                                                                                                                                                                                             |
|--------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `j2_backup_directory`*         | The folder on the host where the backup will be stored. The user must have write permissions.                                                                                                                                                                                           |
| `j2_bin_directory`             | The folder containing the binary and configuration files for the OpenSILEX instance. Leave empty to ignore them in the backup.                                                                                                                                                          |
| `j2_logs_directory`            | The folder containing the logs of the instance. Leave empty to not save the logs.                                                                                                                                                                                                       |
| `j2_data_directory`            | The folder containing the datafiles of the instance stored on the filesystem. Leave empty to not save them.                                                                                                                                                                             |
| `j2_mongo_host`                | The MongoDB host to which the instance is connected (e.g. `localhost:27017`). Leave empty to not save the MongoDB database. You also need to define `j2_mongo_database`.                                                                                                                |
| `j2_mongo_database`            | The MongoDB database to which the instance is connected. This variable is only used if `j2_mongo_host` is defined.                                                                                                                                                                      |
| `j2_rdf4j_host`                | The RDF4J/GraphDB server to which the instance is connected (e.g. `http://localhost:8080/rdf4j-server` or `http://localhost:7200`). Leave empty to not save the RDF repository. You also need to define `j2_rdf4j_repository`.                                                          |
| `j2_rdf4j_repository`          | The RDF4J/GraphDB repository to which the instance is connected. This variable is only used if `j2_rdf4j_host` is defined.                                                                                                                                                              |
| `j2_backup_retaining_duration` | The backup storage periods. Must be in a format compatible with the `date --date` command. For instance, `-30 day` to keep the backups for 30 days. Leave empty to disable the automatic deletion of old backups.                                                                       |
| `j2_remote_backup_login`       | The remote storage server user, used to perform the SSH connexion.                                                                                                                                                                                                                      |
| `j2_remote_backup_host`        | The remote server where the backups will be sent using `rsync`. Leave empty to disable the backup replication. You also need to defined all `j2_remote_*` variables.                                                                                                                    |
| `j2_remote_backup_directory`   | The folder on the remote storage server where the backups will be copied.                                                                                                                                                                                                               |
| `j2_remote_backup_ssh_keyfile` | The path to the SSH private key used by the script to connect to the remote server. If `remote_ssh_key` is defined, its content is copied into this file. Similarly, if `remote_ssh_key_public` is defined, a public key file is also created at this location (with `.pub` extension). |

## Recommendations and useful links

### Use host groups to reuse variables between hosts

In your Ansible inventory, you can use the **host groups** to share some variables across hosts. All the variables
defined for a group are applied to all hosts in this group, but hosts can override them if they need to.

Ansible documentation : [How to build your inventory](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html).

### Protect you secrets with Ansible Vault

The passwords or SSH private keys should not be directly stored in variables. Ansible has a system called Ansible Vault
to encrypt secrets and reference them in variables.

Ansible documentation : [Protecting sensitive data with Ansible vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html).

## Command line

Here are some commonly used commands.

Ansible documentation : [ansible-playbook](https://docs.ansible.com/ansible/latest/cli/ansible-playbook.html).

```shell
# Validate a playbook with a specific inventory with --check
ansible-playbook playbook_deploy_backup.yml -i inventory.yml --check

# Run the playbook on a specific inventory
ansible-playbook playbook_deploy_backup.yml -i inventory.yml

# Restrict the playbook to a group or host using -l (here "phis")
ansible-playbook playbook_deploy_backup.yml -i inventory.yml -l phis

# Ask for the password when using Ansible Vault with --ask-vault-pass
ansible-playbook playbook_deploy_backup.yml -i inventory.yml --ask-vault-pass

# Fetch the Vault password from a file with --vault-password-file
ansible-playbook playbook_deploy_backup.yml -i inventory.yml --vault-password-file vault-ansible-file
```

## Example inventory file

Here is an example of an inventory file for some instances of PHIS.

```yaml
all:
  children:
    phis:
      vars:
        # Host connexion variables
        ansible_user: opensilex
        # Deployment configuration variables
        backup_deploy_location: "/home/{{ ansible_user }}/opensilex-backup"
        backup_log_location: "{{ backup_deploy_location }}/log"
        backup_logrotate_location: "{{ backup_log_location }}/logrotate"
        remote_ssh_key: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          [...]
        remote_ssh_key_public: "ssh-ed25519 [...]"
        # Scripts configuration variables
        j2_backup_directory: "{{ backup_deploy_location }}/backups"
        j2_bin_directory: "/home/{{ ansible_user }}/opensilex-instances/bin/{{ inventory_hostname }}"
        j2_logs_directory: "/home/{{ ansible_user }}/opensilex-instances/logs"
        j2_mongo_host: localhost
        j2_mongo_database: "{{ inventory_hostname }}"
        j2_rdf4j_host: http://localhost:8080/rdf4j-server
        j2_rdf4j_repository: "{{ inventory_hostname }}"
        j2_backup_retaining_duration: "-30 day"
        j2_remote_backup_login: negrev
        j2_remote_backup_host: muse-login.meso.umontpellier.fr
        j2_remote_backup_ssh_keyfile: "/home/{{ ansible_user }}/.ssh/some_key"
        j2_remote_backup_directory: "/home/{{ j2_remote_backup_login }}/opensilex-backup/{{ inventory_hostname }}"
      hosts:
        m3p: # The configuration is identical to the group, no override
        diaphen:
          j2_mongo_host: 10.0.0.135 # Override a variable
        pheno3c:
          j2_mongo_host: 10.0.0.135
        phenotoul:
          j2_mongo_host: 10.0.0.135
        phenotic:
          j2_mongo_host: 10.0.0.135
```

We define a "phis" group, containing the hosts "m3p", "diaphen", "pheno3c", "phenotoul" and "phenotic".

Among the connexion variables, we only define `ansible_user`. That means that Ansible will use the `ssh opensilex@m3p`
command to connect, for example, to the "m3p" host. "m3p" must be defined as a hostname on your machine, for example
in the `.ssh/config` file.

In the deployment variables, we first define `backup_deploy_location`, in which all stuff related to the backup script
will be deployed. We chose a folder located inside the `home` of our user, so that it is guaranteed to have write
permissions.

`backup_log_location` and `backup_logrotate_location` are defined as sub-folders of `backup_deploy_location` to keep
thinks well organized and assert that our user has the correct permissions.

The private key `remote_ssh_key` is encrypted using Ansible Vault, so we will have to specify the vault password when
executing the playbook. However, the public key can be stored without encryption.

As all PHIS instances are configured in a similar way, we can define all script configuration variables in the group.
To reference the hostname, we use the Ansible variable `inventory_hostname`. For example, `j2_mongo_database` will take
the value `m3p` when deploying the host "m3p", and `diaphen` when deploying "diaphen".

The value for `j2_mongo_host` is defined as `localhost` inside the group, however some instances are connected to
a remote MongoDB server. We can override this value by declaring the same variable inside the hosts. When a variable
is declared in the host and the group, the host will take priority.