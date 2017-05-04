# CodeRefinery Gitlab Devops

This repository contains Ansible playbooks for administering the CodeRefinery
gitlab installation. It should not be kept on the gitlab installation to avoid
obvious chicken-egg problems.

### Setting up the environment

This assumes you have virtualenvwrapper, virtualenv and in general Python
installed.

  $ mkvirtualenv cr-gitlab-devops
  (cr-gitlab-devops)$ pip install -r requirements.txt

On subsequent uses it suffices to activate the virtual environment

  $ workon cr-gitlab-devops
  (cr-gitlab-devops)$


## Playbook

There is (will be) a playbook called playbook.yml. The playbook will provision
the necessary resources from OpenStack and configure the system as much as
possible.

It requires certain environment variables for OpenStack authentication.
The password, at least will be distributed separately in this stage. In step 2
a separate user account will be created and an Ansible vault will be
established for secrets.


  (cr-gitlab-devops)$ source project-openrc.sh
  Please enter your OpenStack Password:
  (cr-gitlab-devops)$


Then you can run the actual playbook provided you have the vault password.

  (cr-gitlab-devops)$ ansible-playbook playbook.yml
  Vault password:

#### Ansible vault

To de-crypt the encrypted secrets found in group\_vars/all/vault.yml
one needs a vault password. It is distributed separately.

To view the contents of the vault file run

  (cr-gitlab-devops)$ ansible-vault view group\_vars/all/vault.yml

And to edit it run

  (cr-gitlab-devops)$ ansible-vault edit group\_vars/all/vault.yml

For more information check out  [Ansible vault
documentation](http://docs.ansible.com/ansible/playbooks_vault.html)

## Provisioning

Step one of the playbook creates the following servers

* the actual gitlab host with a public ip
* a separate backup-host with a volume for backups and a cron-job that runs
  backups on gitlab-host
* a gitlab-runner instance that runs gitlab-runner inside docker against the gitlab instance

  * there was a chicken and egg issue with getting the authentication key for
    runner so after installing gitlab you will need to get the files

After the hosts are created the system creates ssh.cfg in the root of your
installation, which is used to use the gitlab installation as bastion so that
the gitlab installation can be the only one of the three items with a public
IP address.

To communicate with the other two parts run

    $ ssh -F ssh.cfg gitlab-runner / gitlab-backup

### Configuring

Most configuration is in group\_vars/all/vars.yml or the vault.yml that was
already covered. Moving per-role variables to the corresponding role is a TODO
item.

Most values should be self-explanatory. If in doubt try grepping for how they
are used.

When adding a variable to the vault make a line in vars.yml that copies the
variable to another variable without the prefix vaulted\_ . This makes it
easier to check the names of variables in the vault without decrypting it all the time..

Initial installation root password is stored inside the vault. Check that to
log in and start managing the server if installing from scratch.

### SSL

SSL keys were created by using certbot. There is a part called letsencrypt
that will in theory re-generate the certificates using a brand new ansible
module. This is doubtable in practice and someone should re-generate the
certificates manually before summer holidays 2017.

The playbook for letsencrypt is included but disabled in case there is time to
work on it.
