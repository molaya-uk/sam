# App Manager
An Ansible based framework for managing app deployment for Splunk.

## Introduction
The project contains a simple Ansible based framework that can be used to manage apps and add-ons for Splunk deployments. The main goal of the framework is to provide a simple, flexible and extendable solution that can be easily adjusted for a particular platform.

Apps and add-ons are kept in a Git repository. The playbook pulls the repository to the Ansible server, copies each app to the Splunk tiers listed for it in a manifest, renders any app specific templates, and applies the configuration on each tier.

## Features
- **App Manifest** - all app and add-on deployment activities are defined within a single manifest file.
- **Templates and variables kept with the apps** - each app can contain a templates directory and a variables file specific to that app, so templates and their variables are not kept separately from the apps.
- **Secrets from a secrets manager** - template variables can be looked up from a secrets manager (or Ansible Vault) when the templates are rendered, so no secrets are kept in the apps repository.
- **One play per Splunk tier** - the deployer, cluster master, deployment server and etc/apps are managed by separate plays, each followed by the right apply step.
- **Easy to extend** - all tier plays share the same tasks and differ only in a few variables, so adding an environment (e.g. Production and Pre-production) or a new destination is a short play.

## Contents
- [How it works](#how-it-works)
- [Repository layout](#repository-layout)
- [Requirements](#requirements)
- [Configuration](#configuration)
- [Manifest](#manifest)
- [App structure](#app-structure)
- [App variables](#app-variables)
- [Worked example](#worked-example)
- [Running the playbook](#running-the-playbook)
- [Extending the playbook](#extending-the-playbook)
- [Security](#security)
- [Known limitations](#known-limitations)

## How it works
The playbook (main.yml) contains the following plays:

| Play tag | Inventory group | Apps directory | Deploys apps tagged | Clear step | Apply step |
|---|---|---|---|---|---|
| repo_pull | localhost | - | - | - | Pulls the apps repository to *git_repo_destination* |
| deployer | deployer | etc/shcluster/apps | deployer, all | clear_apps.yml | `splunk apply shcluster-bundle` |
| master | cluster_master | etc/master-apps | master, all | clear_master.yml | `splunk apply cluster-bundle` |
| deployment | deployment_server | etc/deployment-apps | deployment, all | clear_apps.yml | `splunk reload deploy-server` |
| etc | etc_apps | etc/apps | etc_apps, all | clear_etc_apps.yml | `splunk restart` |

Each tier play runs tasks/deploy.yml, which:
1. Loads the environment file passed in *template_definition*, if one is given.
2. Empties the apps directory (when *remove_before_copying* is true):
   - clear_apps.yml removes everything.
   - clear_master.yml keeps the `_cluster` app, unless *remove_cluster_app* is true.
   - clear_etc_apps.yml keeps the apps listed in *splunk_default_apps*.
3. Runs tasks/apps.yml for every manifest entry tagged for the play. For each app it:
   1. copies the app from the Git checkout,
   2. loads the app's variables file into *app_vars*,
   3. renders the app's templates into its local/ or default/ directory,
   4. removes the templates directory and the variables file from the copied app.
4. Applies the configuration.

## Repository layout
```
main.yml                    Playbook with the repo pull play and one play per Splunk tier
header.yml                  Global settings
manifest.yml                Example manifest
production.yml              Example environment file (passed as template_definition)
hosts                       Example inventory
tasks/deploy.yml            Steps shared by all tier plays
tasks/apps.yml              Deploys a single app
tasks/clear_apps.yml        Empties an apps directory
tasks/clear_master.yml      Empties master-apps, optionally preserving _cluster
tasks/clear_etc_apps.yml    Empties etc/apps, preserving Splunk default apps
tasks/deployer_apply.yml    Applies the search head cluster bundle
tasks/master_apply.yml      Applies the indexer cluster bundle
tasks/deployment_reload.yml Reloads the deployment server
tasks/etc_restart.yml       Restarts Splunk
```

## Requirements
- Ansible on the Ansible server (2.4 or later, for `include_tasks`).
- Access from the Ansible server to the Git repository containing the apps, using SSH keys or a URL with credentials.
- SSH access from the Ansible server to the Splunk servers, with the ability to become the Splunk user.
- Any Ansible collections that provide the lookups used in app variables files, installed on the Ansible server, e.g.:
  ```
  ansible-galaxy collection install amazon.aws community.hashi_vault
  ```

## Configuration
### header.yml
| Variable | Default | Description |
|---|---|---|
| git_repo_location | `https://repo.local/apps.git` | Git repository containing apps and add-ons |
| git_version | `HEAD` | Branch, tag or commit to deploy |
| git_repo_destination | `/tmp/splunk_apps` | Directory on the Ansible server the repository is pulled to |
| localhost_user | `vagrant` | User pulling the repository on the Ansible server |
| use_repo | `true` | Set to false to skip pulling the repository and deploy the existing checkout |
| template_directory | `templates` | Name of the templates directory in each app |
| app_variables_file | `variables.yml` | Name of the variables file in each app |
| remote_ansible_user | `vagrant` | User connecting to the Splunk servers |
| remote_become | `true` | Whether to become another user on the Splunk servers |
| remote_become_user | `splunk` | User to become on the Splunk servers |
| ansible_tmp | `{{ splunk_home }}/ansible` | Directory default apps are backed up to while etc/apps is emptied |
| remove_before_copying | `true` | Empty the apps directory before copying apps |
| remove_cluster_app | `false` | Also remove `master-apps/_cluster` when emptying master-apps |
| splunk_home | `/opt/splunk` | Splunk installation directory |
| splunk_default_apps | (list) | Apps shipped with Splunk that are preserved when emptying etc/apps |
| preserve_lookups | `"true"` | `-preserve-lookups` value for `apply shcluster-bundle` |
| shcluster_bundle_target | `https://sh1:8089` | Search head cluster member targeted by `apply shcluster-bundle` |
| splunk_user | `admin` | Splunk user used by the apply and reload commands |
| splunk_password | `foo` | Password of *splunk_user*; see [Security](#security) |

Check *splunk_default_apps* against `ls $SPLUNK_HOME/etc/apps` on a fresh installation of your Splunk version. Apps that are not in the list are deleted from etc/apps by the etc play. Names in the list that do not exist on a host are skipped.

### Inventory
The plays target the following inventory groups:

```ini
[deployer]
deployer1

[cluster_master]
cm1

[deployment_server]
ds1

[etc_apps]
hf1
hf2
```

A host can be a member of several groups. A play whose group is missing from the inventory, or empty, is skipped.

### Environment file
An environment file contains variables for a particular environment and is passed with `-e template_definition=<file>`. Its variables are available to templates and to app variables files, e.g. production.yml:

```yaml
environment_name: production
hec_default_index: prod_main
```

## Manifest
**manifest.yml** defines which apps are deployed and where. Each app managed by the playbook requires an entry:

```yaml
manifest:
  - description: HEC inputs for application logs
    src: inputs/app_hec_inputs
    dest: app_hec_inputs
    server_tags: ['deployment']
    templates: ['inputs']
    template_dest: local
```

| Attribute | Required | Description |
|---|---|---|
| description | no | Information on the app; not used by the playbook |
| src | yes | Path to the app relative to *git_repo_destination* |
| dest | yes | Name of the app on the Splunk servers |
| server_tags | yes | Tags selecting the plays the app is deployed by, e.g. `deployer`, `master`, `deployment`, `etc_apps` or `all` |
| templates | yes | Template names found under `<app>/<template_directory>/`; `[]` if the app has no templates |
| template_dest | when templates are listed | `local` or `default`; the directory the templates are rendered to |

A template named `inputs` is read from `<app>/templates/inputs.j2` and rendered to `<app>/<template_dest>/inputs.conf`. A rendered template replaces a file with the same name copied from the repository.

Using `local` as *template_dest* keeps the app's default/ directory as it is in the repository.

### Manifest examples
An app deployed to all tiers, without templates:

```yaml
  - description: Organisation wide props and transforms
    src: org_all_props
    dest: org_all_props
    server_tags: ['all']
    templates: []
```

An app deployed to the search head cluster and indexer cluster:

```yaml
  - description: Technology add-on for Linux
    src: addons/Splunk_TA_nix
    dest: Splunk_TA_nix
    server_tags: ['deployer', 'master']
    templates: []
```

An app deployed under a different name than in the repository:

```yaml
  - description: Search head outputs, deployed as org_sh_outputs
    src: outputs/search_head
    dest: org_sh_outputs
    server_tags: ['deployer']
    templates: []
```

An app with two templates rendered to local/:

```yaml
  - description: Forwarder outputs with TLS settings
    src: outputs/forwarders
    dest: org_fwd_outputs
    server_tags: ['deployment']
    templates: ['outputs', 'server']
    template_dest: local
```

## App structure
An app in the repository can contain, in addition to its Splunk files, a templates directory and a variables file:

```
app_hec_inputs
|- default/app.conf
|- local/props.conf
|- templates/inputs.j2
|- variables.yml
```

Both are optional and both are removed from the app after the templates are rendered, so they are not deployed to Splunk. Their names are set by *template_directory* and *app_variables_file* in header.yml.

Templates are Jinja templates rendered with the Ansible template module. They can use:
- the app's variables, under *app_vars*, e.g. `{{ app_vars.hec_token }}`,
- variables from header.yml, the environment file, the inventory and `-e`, e.g. `{{ environment_name }}`.

## App variables
The variables file contains the variables required by the app's templates. It is loaded before the app's templates are rendered, and its variables are available under *app_vars*. Variables of one app are not available to templates of other apps. If an app has no variables file, *app_vars* is empty.

The values in the variables file are evaluated on the Ansible server when a template uses them. A value can therefore be a lookup that obtains it, for example from a secrets manager. Lookups run as the user running `ansible-playbook`, using that user's credentials for the secrets manager.

### Plain values
```yaml
hec_name: app_logs
hec_index: app_main
hec_sourcetype: app:json
```

### Values depending on the environment
Variables from the environment file can be used in values:

```yaml
hec_index: "{{ environment_name }}_app"
indexer_url: "{{ 'https://idx.prod.local:8088' if environment_name == 'production' else 'https://idx.dev.local:8088' }}"
```

### Default values
```yaml
hec_index: "{{ hec_default_index | default('main') }}"
```

### AWS Secrets Manager
Requires the `amazon.aws` collection:

```yaml
hec_token: "{{ lookup('amazon.aws.secretsmanager_secret', 'splunk/' ~ environment_name ~ '/hec_token') }}"
```

### HashiCorp Vault
Requires the `community.hashi_vault` collection. Vault connection settings are taken from the usual `VAULT_ADDR` and `VAULT_TOKEN` environment variables:

```yaml
hec_token: "{{ lookup('community.hashi_vault.vault_kv2_get', 'splunk/' ~ environment_name ~ '/hec', engine_mount_point='secret').secret.token }}"
```

### Ansible Vault
Values encrypted with `ansible-vault encrypt_string` can be stored in the variables file directly:

```
ansible-vault encrypt_string 'my-hec-token' --name hec_token
```

```yaml
hec_token: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          62313365396662343061393464336163383764373764613633653634306231386433626436623361
          ...
```

The playbook must then be run with `--ask-vault-pass` or `--vault-password-file`.

### Environment variables and files on the Ansible server
```yaml
hec_token: "{{ lookup('env', 'SPLUNK_HEC_TOKEN') }}"
ssl_password: "{{ lookup('file', '/etc/ansible/secrets/splunk_ssl_password') }}"
```

## Worked example
This example deploys an app with a HEC input to the deployment server, using a token kept in AWS Secrets Manager.

Repository layout:

```
apps.git
|- inputs/app_hec_inputs
   |- default/app.conf
   |- templates/inputs.j2
   |- variables.yml
```

inputs/app_hec_inputs/variables.yml:

```yaml
hec_name: app_logs
hec_index: "{{ environment_name }}_app"
hec_token: "{{ lookup('amazon.aws.secretsmanager_secret', 'splunk/' ~ environment_name ~ '/hec_token') }}"
```

inputs/app_hec_inputs/templates/inputs.j2:

```
[http://{{ app_vars.hec_name }}]
token = {{ app_vars.hec_token }}
index = {{ app_vars.hec_index }}
disabled = 0
```

manifest.yml entry:

```yaml
  - description: HEC inputs for application logs
    src: inputs/app_hec_inputs
    dest: app_hec_inputs
    server_tags: ['deployment']
    templates: ['inputs']
    template_dest: local
```

production.yml:

```yaml
environment_name: production
```

Invocation:

```
ansible-playbook -i hosts -e template_definition=production.yml main.yml --tags repo_pull,deployment
```

Result on the deployment server, in `$SPLUNK_HOME/etc/deployment-apps/app_hec_inputs`:

```
app_hec_inputs
|- default/app.conf
|- local/inputs.conf
```

local/inputs.conf:

```
[http://app_logs]
token = <value of splunk/production/hec_token>
index = production_app
disabled = 0
```

## Running the playbook
Load the SSH keys for Git and the Splunk servers and run the playbook:

```
ssh-agent bash
ssh-add git_key
ssh-add remote_server_key
ansible-playbook -i hosts -e template_definition=production.yml main.yml
```

Deploy to selected tiers only, using the play tags. Include `repo_pull` unless the checkout in *git_repo_destination* is already up to date:

```
ansible-playbook -i hosts -e template_definition=production.yml main.yml --tags repo_pull,deployer,master
```

Deploy a specific branch or commit:

```
ansible-playbook -i hosts -e template_definition=production.yml -e git_version=release-2.3 main.yml
```

Deploy to selected hosts only:

```
ansible-playbook -i hosts -e template_definition=production.yml main.yml --tags repo_pull,etc --limit hf1
```

Use variables or files encrypted with Ansible Vault, e.g. the Splunk credentials (see [Security](#security)):

```
ansible-playbook -i hosts -e template_definition=production.yml -e @secrets.yml --ask-vault-pass main.yml
```

Check mode (`--check`) skips the clear, apply and restart steps and reports the files that would be copied or rendered. It does not report files that would be deleted.

## Extending the playbook
Each tier play sets the following variables, used by tasks/deploy.yml:

| Variable | Description |
|---|---|
| app_tags | The *server_tags* selecting the apps deployed by the play |
| apps_dir | The directory apps are copied to |
| clear_tasks | The task file in tasks/ emptying *apps_dir* before copying |
| apply_tasks | The task file in tasks/ applying the configuration |

### Multiple environments
For example, to manage two indexer clusters (Production and Pre-production) via master-apps, replace the master play with a play per cluster, each with its own inventory group, play tag and *app_tags*.

Default play:

```yaml
- hosts: cluster_master
  tags: [ master ]
  remote_user: "{{ remote_ansible_user }}"
  become: "{{ remote_become }}"
  become_user: "{{ remote_become_user }}"
  vars_files:
  - header.yml
  - manifest.yml
  vars:
    app_tags: ['master', 'all']
    apps_dir: "{{ splunk_home }}/etc/master-apps"
    clear_tasks: clear_master.yml
    apply_tasks: master_apply.yml
  tasks:
  - include_tasks: "tasks/deploy.yml"
```

Modified plays:

```yaml
- hosts: production_cluster_master
  tags: [ prod_master ]
  remote_user: "{{ remote_ansible_user }}"
  become: "{{ remote_become }}"
  become_user: "{{ remote_become_user }}"
  vars_files:
  - header.yml
  - manifest.yml
  vars:
    app_tags: ['prod_master', 'prod_all']
    apps_dir: "{{ splunk_home }}/etc/master-apps"
    clear_tasks: clear_master.yml
    apply_tasks: master_apply.yml
  tasks:
  - include_tasks: "tasks/deploy.yml"

- hosts: pre_cluster_master
  tags: [ pre_master ]
  remote_user: "{{ remote_ansible_user }}"
  become: "{{ remote_become }}"
  become_user: "{{ remote_become_user }}"
  vars_files:
  - header.yml
  - manifest.yml
  vars:
    app_tags: ['pre_master', 'pre_all']
    apps_dir: "{{ splunk_home }}/etc/master-apps"
    clear_tasks: clear_master.yml
    apply_tasks: master_apply.yml
  tasks:
  - include_tasks: "tasks/deploy.yml"
```

Manifest entries then use the new tags:

```yaml
  - description: Indexes for both clusters
    src: org_all_indexes
    dest: org_all_indexes
    server_tags: ['prod_master', 'pre_master']
    templates: []
```

Different values per environment are provided by passing a different environment file, e.g. `-e template_definition=preprod.yml`, and using its variables in the app variables files.

### A new destination
For example, to manage heavy forwarders separately from other etc/apps hosts, add an inventory group and a play with its own tag:

```yaml
- hosts: heavy_forwarders
  tags: [ hf ]
  remote_user: "{{ remote_ansible_user }}"
  become: "{{ remote_become }}"
  become_user: "{{ remote_become_user }}"
  vars_files:
  - header.yml
  - manifest.yml
  vars:
    app_tags: ['hf', 'all']
    apps_dir: "{{ splunk_home }}/etc/apps"
    clear_tasks: clear_etc_apps.yml
    apply_tasks: etc_restart.yml
  tasks:
  - include_tasks: "tasks/deploy.yml"
```

### Custom clear and apply steps
*clear_tasks* and *apply_tasks* can name any task file in tasks/. The tasks run on the play's hosts as *remote_become_user* and can use the play variables, including *apps_dir*. For example, tasks/clear_none.yml, for a destination that should not be emptied:

```yaml
- name: keep existing apps
  debug:
    msg: "Apps in {{ apps_dir }} are not removed"
```

Splunk 9.0 renamed master-apps to manager-apps. If your cluster manager uses manager-apps, set *apps_dir* to `{{ splunk_home }}/etc/manager-apps`.

## Security
- Do not keep *splunk_password* in header.yml in plain text. Move *splunk_user* and *splunk_password* to a file encrypted with Ansible Vault and pass it with `-e @secrets.yml --ask-vault-pass`. Extra variables override the values in header.yml.
  ```
  ansible-vault create secrets.yml
  ```
  ```yaml
  splunk_user: admin
  splunk_password: <password>
  ```
- The apply and reload tasks pass the credentials to the Splunk CLI with `-auth`. They use `no_log`, so the credentials are not shown in the Ansible output. As a result, errors from these commands are not shown either; set `no_log: false` temporarily to troubleshoot. The credentials are visible in the process list on the Splunk server while a command runs.
- Rendered templates may contain secrets. They are shown in the Ansible output when running with `--diff`, and in the error output if a template task fails.
- Lookups in app variables files run once for each template that uses the value, on each host. With many apps or hosts this results in many requests to the secrets manager.

## Known limitations
- The playbook always empties and recopies the apps directories, and runs the apply step on every run, whether or not any app changed. The etc play restarts Splunk on every run.
- When emptying etc/apps, apps that are not listed in *splunk_default_apps* are deleted, including apps installed through Splunk Web.
- etc/apps is emptied before the default apps are restored. If the run fails between these steps, etc/apps is left without the default apps, and the copies in *ansible_tmp* are restored by the next run.
