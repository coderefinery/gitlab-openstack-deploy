# CodeRefinery Gitlab Devops

This repository contains Ansible playbooks for administering the CodeRefinery
gitlab installation. It should not be kept on the gitlab installation to avoid obvious chicken-egg problems.


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


### Provisioning

The playbook called provisioning.yml will provision the required hardware. In
step 1 this consists of

* docker host

* ports:
  one with public ip for the instance
  another with a public ip for mainenance

* security group for the host
 * permit port 22 to the host
In step two there will be 1-N of

* gitlab-ci-multi-runner
  * unsure if in a separate VM, container or what


### Configuring/installing

On docker host
 * install docker, other possible prerequisites
 * mount extra volume at /srv/docker/gitlab/gitlab
    * chcon the directory for container access
 * pull {{ new_version }} image(s) for docker, postgres and redis
 * if {{ old_versin != new_version }} stop and remove {{ old_version }} image
  * optional backup here if needed
  * ensure postgres and redis are started
  * pull and start {{ new_version }} gitlab container

Upgrading:

edit new_version to be 1 above old_version, run again.


