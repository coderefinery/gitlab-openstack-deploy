# Gitlab Devops

This repository contains Ansible playbooks for administering the gitlab
installations on OpenStack.. It should not be kept on the gitlab installation
to avoid obvious chicken-egg problems.

### Setting up the Python environment

This assumes you have virtualenvwrapper, virtualenv and in general Python
installed.

  $ mkvirtualenv cr-gitlab-devops
  (cr-gitlab-devops)$ pip install -r requirements.txt

On subsequent uses it suffices to activate the virtual environment

  $ workon cr-gitlab-devops
  (cr-gitlab-devops)$

### Setting up environment variables

This repository does not contain per-installation data.

The data is expected to be found thus

  repo/
    playbook.yml
    roles/...
    environment/ # here you can have multiple environments
      coderefinery-gitlab/ # symlink to a separate repo
        hosts
        group_vars/
          all/
            vars.yml
            vault.yml
            ansible.cfg
            other.yml
      another-gitlab/
        hosts
        group_vars
          all/
            vars.yml
            vault.yml
            something_completely_different.yml

To create a new set of environment variables it is suggested to copy an
existing set for simplicity.

## Playbook

There roles are run with playbook called playbook.yml. The playbook will provision
the necessary resources from OpenStack and configure the system as much as
possible.

It requires certain environment variables for OpenStack authentication called
an OpenRC.

  (cr-gitlab-devops)$ source project-openrc.sh
  Please enter your OpenStack Password:
  (cr-gitlab-devops)$

Then you can run the actual playbook provided you have the vault password.

  (cr-gitlab-devops)$ export ANSIBLE\_CONFIG=~/path/to/coderefinery-gitlab/ansible.cfg
  ansible-playbook playbook.yml -i environments/coderefinery-gitlab/hosts
  Vault password:

and Bob is your uncle!

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

    $ ssh -F ssh.{{gitlab_name}}.cfg gitlab-runner / gitlab-backup

### Configuring

Most configuration is in group\_vars/all/vars.yml or the vault.yml that was
already covered.

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

### Recovering backups

1) obtain files and copy them to remote machine, e.g.

        [cloud-user@gitlab-internal tmp]$ ls -alhZ
        drwxrwxrwt. root       root       system_u:object_r:tmp_t:s0       .
        drwxr-xr-x. root       root       system_u:object_r:root_t:s0      ..
        -rw-------. cloud-user cloud-user unconfined_u:object_r:user_tmp_t:s0
        1495065614_2017_05_18_gitlab_backup.tar
        -rw-------. cloud-user cloud-user unconfined_u:object_r:user_tmp_t:s0
        etc-gitlab-1495065602.tgz

2)  copy  xxxx_gitlab_backup.tar to /srv/gitlab/data/backups
    chmod 0755 /srv/gitlab/data/backups/xxx_gitlab_backup.tar

  copy etc-gitlab-XXX.tgz to /srv/gitlab/config/backups/ (optional, you can
  pack it wherever you like)

3) shutdown stuff that uses the database

   docker exec -it gitlab gitlab-ctl stop unicorn
   docker exec -it gitlab gitlab-ctl stop sidekiq

4) unpack the config .tgz, copy files you wish to replace (everything except
gitlab.rb probably, unless you changed certificates)

5) run

   docker exec -it gitlab gitlab-rake gitlab:backup:restore \
   BACKUP=1495065614_2017_05_18

with the timestamp of the backup. Answer YES to everything.
