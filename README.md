# RHEL 8 demo Red Hat Forum 2019



This is a setup and run guide for the Red Hat Enterprise Linux 8 demo presented at Red Hat Forum in the Nordics 2019.

Short description:

Demonstrate how a base image can be deployed in an automated way to public cloud, including configuration and the use of Red Hat Insights to evaluate security.



## RHEL8 virutal machine with composer

**System requirements:**

* RHEL 8 virtual machine
* RAM: 4GB (8GB recommended)
* CPU: 2
* A running instance of Ansible Tower
* Public cloud credentials or local environment to deploy images to


**Install:** 

If there is intention to show OpenSCAP policies with SCAP-Workbench, do a standard install with GUI. If that is not required, basic minimal install.

**Post install:**

`subscription-manager register`

`subscription-manager attach --pool=<poolID>`

Create rheldemo user and generate ssh key-pair:

`useradd rheldemo`

`passwd rheldemo`(use rheldemo as password)

`usermod -aG wheel rheldemo`

`sudo su - rheldemo`

`ssh-keygen`

The public key stored in /home/rheldemo/.ssh/id_rsa.pub should your composer-toml file, and the private key stored in /home/rheldemo/.ssh/id_rsa should be added as a credential in Ansible Tower.

Enable and start Cockpit:

`systemctl enable --now cockpit.socket`

Install Composer:

`yum install -y lorax-composer composer-cli cockpit-composer`

Start Composer:

`systemctl enable --now lorax-composer.service`

If intention is to later in demo show OpenSCAP policy in GUI, install required components:

`yum install -y scap-security-guide openscap-scanner scap-workbench`


## Building an image with Composer
Logon to Cockpit UI and select the Composer plugin.

Create a new blueprint and select the following packages:

* openssh
* openssh-server
* insights-client
* openscap-scanner
* scap-security-guide
* platform-python

### Customization of composer blueprint:
Export blueprint:
`composer-cli blueprints save <blueprint name>`

Edit blueprint and add the following text at the end: (password = rheldemo + insert your ssh key)

```
[customizations]

[[customizations.user]]
name = "rheldemo"
description = "Red Hat Nordics"
password = "$6$qgtDGzzlmyqGnCOB$FA58kD2VZ1lmPB3rYib7jtH0a5tByi2Kwqp29Zw7rySqV.jnMAK9qbWUZ.OsDvO/jljWActCC9pmBiXBRZdAa/"
key = "<insert public ssh-key>"
home = "/home/rheldemo/"
shell = "/usr/bin/bash"
groups = ["users", "wheel"]

[customizations.services]
enabled = ["sshd", "cockpit.socket", "httpd"]
disabled = ["postfix", "telnetd"]
```

Push changes back to Composer:
`composer-cli blueprints push <name_of_blueprint.toml>`

Create desired images in Composer UI based on new blueprint.

**Troubleshooting:**
Composer logs are stored under /var/log/lorax-composer

If issue in Composer GUI with inventory not loading:
`systemctl restart lorax-composer.service`

### Post install tasks (done with Ansible playbooks/roles):
`subscription-manager register`

`subscription-manager attach --pool=<poolID>`

Assign and remediate OpenSCAP OSPP security policy:

`oscap xccdf eval --remediate --profile xccdf_org.ssgproject.content_profile_ospp --results scan.xml /usr/share/xml/scap/ssg/content/ssg-rhel8-ds.xml`

`insights-client --register`

`insights-client --verbose --payload scan.xml --content-type application/vnd.redhat.compliance.something+tgz`
