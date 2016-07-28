# COLDSLEEP

## PUPPET

**Befehle:**

Puppet Master in Foreground:

	puppet master --no-daemonize --verbose

Puppet Agent Run in Foreground:

	puppet agent --test --server=puppet --noop --verbose --debug
	
	puppet agent --test --server=puppet --verbose --environment=production

Parse and print out the effective Puppet.conf File:

	puppet config print

... parameter *modulepath*:

	puppet config print modulepath --section master --environment production
	puppet config print modulepath --section master --environment development
	puppet config print modulepath --section master --environment testing
	puppet config print modulepath --section master --environment feature1
	...

	puppet config print modulepath --section agent --environment production

... parameter *manifest*:

	puppet config print manifest --section master --environment production
	...

**Allgemein:**

*puppet.conf:*
	$confdir = /etc/puppet

**Map of Puppet Enviroments to git branches:**

|Puppet Enviroment|Git Branch|
|-----------------|:---------|
|production|production|
|development|development|
|testing|testing|

Applies to both:
- puppet modules
- puppet projects

*puppet.conf/[master]:*

	environmentpath = $confdir/environments

**Dynamic Puppet Enviroments with Git Branches:**

[Git Repository of the Puppet Project, Web Hooks (post-receive hook)](http://gitlab.hochguertel.biz/puppet-projekte/hochguertel-biz-infrastructure/hooks)


**Workflow:**

1. Clone the puppet project repository.
2. Make changes, and then push a testing branch named feature1.
3. The Git Server fires off a post-receive hook that takes the following action:
	1. Create a new folder `/etc/puppet/enviroments/feature1`.
	2. Perform a Git Clone of the central repository in the folder feature1.
	3. Check out the branch feature1.
4. Go to a test machine and issue `puppet --test --environment=feature1`.
5. Repeat steps 2-4 until good code is produced.
6. Merge the branch feature1 into the master, and push it to the Git server.
7. The Git server fires off the same post-receive hook, which now detects it is updating an environment and performs only an update.

**The key optiomization is that all Git branches that aren't master/production are testing branches. There is no time spent moving a code base into a testing environment; it's automatically put into one whenever you push.** *- Pro-Puppet Book, page 93.*

**Git Hooks:**

**post-receive hooks:**

*dynamic enviroments*

**1) Install, Configurate and Testing:**

*perspective* -> Git Repository Server:

	mkdir -p /tmp/tempt
	cd /tmp/tempt
	git clone https://github.com/pdxcat/puppet-sync.git
	
	cd /home/git/repositories/puppet-projekte/hochguertel-biz-infrastructure.git/
	cp /tmp/tempt/puppet-sync/post-receive hooks/post-receive
	chown git:git hooks/post-receive
	chmod 755 hooks/post-receive

	sudo -u git ssh-keygen -t rsa -b 4096 -f ~git/.ssh/id_rsa
	ssh -l root puppet.hochguertel.biz 'cat - >> ~puppet/.ssh/authorized_keys' < ~git/.ssh/id_rsa.pub
	echo "sudo -u git ssh -l puppet puppet.hochguertel.biz"

	rm -rf /tmp/tempt/

*hooks/post-receive:*

	#!/bin/sh
	# File: /home/git/puppet-projekte/hochguertel-biz-infrastructure.git/hooks/post-receive
	
	REPO="git@gitlab.hochguertel.biz:puppet-projekte/hochguertel-biz-infrastructure.git"
	DEPLOY="/etc/puppet/environments"
	SSH_ARGS="-i /home/git/.ssh/id_rsa"
	PUPPETMASTER="puppet@puppet.hochguertel.biz"
	SYNC_COMMAND="/usr/bin/puppet-sync"
	
	while read oldrev newrev refname
	do
	  BRANCH=`echo $refname | sed -n 's/^refs\/heads\///p'`
	  [ "$newrev" -eq 0 ] 2> /dev/null && DELETE='--delete' || DELETE=''
	
	  ssh $SSH_ARGS "$PUPPETMASTER" "$SYNC_COMMAND" \
	    --branch "$BRANCH" \
	    --repository "$REPO" \
	    --deploy "$DEPLOY" \
	    $DELETE
	
	done

*perspective* -> Puppetmaster Server:

	mkdir -p /tmp/tempt
	cd /tmp/tempt
	git clone https://github.com/pdxcat/puppet-sync.git

	cp /tmp/tempt/puppet-sync/puppet-sync /usr/bin/puppet-sync
	chmod 755 /usr/bin/puppet-sync

	chown -R puppet:puppet /etc/puppet/environment
	usermod -s /bin/sh puppet
	sudo -u puppet ssh-keygen -t rsa -b 4096 -f ~puppet/.ssh/id_rsa
	cat ~puppet/.ssh/id_rsa.pub
	
	echo "Copy this Public SSH Key to the git-user puppet."  
	
	chmod 777 /tmp/tempt/
	cd /tmp/tempt/
	sudo -u puppet git clone git@gitlab.hochguertel.biz:puppet-projekte/hochguertel-biz-infrastructure.git
	cd ~
	rm -rf /tmp/tempt/
	
*perspective* -> Developer, Testing the dynamic enviroment creating hook on branches:

1. *Create branch & Push branch*

	git checkout -b testing_environment
	git push origin testing_environment

2. *perspective* -> Puppet Node:

	puppet agent --test --server=puppet --verbose --environment=testing_environment

3. *Delete local branch & Delete remote branch* 

	git branch -d testing_environment
	git push origin :testing_environment

**2) Renaming branch "master" to "production"**

*perspective* -> Git Repository Server:

	cd /home/git/repositories/puppet-projekte/hochguertel-biz-infrastructure.git/
	echo "ref: refs/heads/production" > HEAD

*perspective* -> Developer:
[howto...](http://stackoverflow.com/questions/8762601/how-do-i-rename-my-git-master-branch-to-release "howto...")

	git checkout -b production master	      
	git push -u origin production        
	git branch -d master              
	git push --delete origin master   
	git remote prune origin

**PuppetDB:**

- https://github.com/puppetlabs/puppetlabs-puppetdb.git
- https://docs.puppetlabs.com/puppetdb/

*perspective* -> Developer:

	git subtree add --prefix modules/standards git@gitlab.hochguertel.biz:puppet-modules/hochguertel-standards.git master --squash
	git push origin production

	git subtree add --prefix modules/puppetdb https://github.com/puppetlabs/puppetlabs-puppetdb.git master --squash
	git push origin production

	git subtree add --prefix modules/inifile https://github.com/puppetlabs/puppetlabs-inifile.git master --squash
	git push origin production
	
	git subtree add --prefix modules/postgresql https://github.com/puppetlabs/puppet-postgresql.git master --squash
	git push origin production
	
	git subtree add --prefix modules/stdlib https://github.com/puppetlabs/puppetlabs-stdlib.git master --squash
	git push origin production
	
	git subtree add --prefix modules/apt https://github.com/puppetlabs/puppetlabs-apt.git master --squash
	git push origin production
	
	git subtree add --prefix modules/concat https://github.com/puppetlabs/puppetlabs-concat.git master --squash
	git push origin production
	
	git subtree add --prefix modules/firewall https://github.com/puppetlabs/puppetlabs-firewall.git master --squash
	git push origin production

	git subtree pull --prefix modules/standards git@gitlab.hochguertel.biz:puppet-modules/hochguertel-standards.git master --squash
	git push origin production

	puppet agent --test --server=puppet --verbose --noop

	git subtree add --prefix modules/ssh https://github.com/saz/puppet-ssh.git master --squash
	git push origin production
	

**IPv6 Configuration:**

[DomainFactory Jiffybox Netzwerk Konfiguration](http://www.df.eu/de/service/df-faq/cloudserver/control-panel/netzwerk-konfiguration/#c9043 "DomainFactory Jiffybox Netzwerk Konfiguration")
![C:\Users\hochguertelto\Desktop\2015-03-24 17_18_06-JiffyBox _ Netzwerk-Konfiguration.png](http://i.imgur.com/WaVKOU7.png)

	ifconfig eth0 inet6 add 2a00:1158:3::28f/64
	ip -6 route add default dev eth0 via fe80::1
	ping6 ipv6.df.eu

**Testing New Puppet Modules:**

*perspective* -> Developer:

	cd /cygdrive/d/DATA/Puppet/1_Projekte/hochguertel.biz/Entwicklung/hochguertel-biz-infrastructure
	git checkout -b promp
	
	git subtree pull --prefix modules/standards git@gitlab.hochguertel.biz:puppet-modules/hochguertel-standards.git master --squash

	git push origin promp

*perspective* -> Puppet Node:

	puppet agent --test --server=puppet --verbose --noop --environment=profileDOTd

**Apache:**

	git subtree add --prefix modules/apache https://github.com/puppetlabs/puppetlabs-apache master --squash
	git push origin production

	git subtree add --prefix modules/staging https://github.com/nanliu/puppet-staging.git master --squash
	git push origin production

	git subtree add --prefix modules/tomcat https://github.com/puppetlabs/puppetlabs-tomcat.git master --squash
	git push origin production

	git subtree add --prefix modules/java https://github.com/puppetlabs/puppetlabs-java.git master --squash
	git push origin production
	
	
	git subtree add --prefix modules/nexus https://github.com/hubspotdevops/puppet-nexus.git master --squash
	git push origin production

	git subtree add --prefix modules/wget https://github.com/maestrodev/puppet-wget.git master --squash
	git push origin production

	git subtree add --prefix modules/zypprepo https://github.com/deadpoint/puppet-zypprepo.git master --squash
	git push origin production

	git subtree add --prefix modules/jenkins https://github.com/jenkinsci/puppet-jenkins master --squash
	git push origin production
	




	git subtree add --prefix modules/rbenv https://github.com/alup/puppet-rbenv.git master --squash
	git push origin production

	git subtree add --prefix modules/vcsrepo https://github.com/puppetlabs/puppetlabs-vcsrepo.git master --squash
	git push origin production

	git subtree add --prefix modules/git https://github.com/puppetlabs/puppetlabs-git.git master --squash
	git push origin production
	
	git subtree add --prefix modules/gitlab https://github.com/sbadia/puppet-gitlab.git master --squash
	git push origin production

	git subtree add --prefix modules/nexus_rest https://bitbucket.org/atlassian/puppet-module-nexus_rest.git master --squash
	git push origin production


	

	git subtree add --prefix modules/ntp https://github.com/puppetlabs/puppetlabs-ntp.git master --squash
	git push origin production

	git subtree add --prefix modules/mysql https://github.com/puppetlabs/puppetlabs-mysql.git  master --squash
	git push origin production


	

**Profile & Role:**

*See: /cygdrive/d/DATA/Puppet/1_Projekte/hochguertel.biz/Entwicklung/#_Modules/puppet-module-role*

*See: /cygdrive/d/DATA/Puppet/1_Projekte/hochguertel.biz/Entwicklung/#_Modules/puppet-module-profiles*

*See: /cygdrive/d/DATA/Puppet/1_Projekte/hochguertel.biz/Entwicklung/#_Modules/puppet-code-examples/london-fundamentals-0613-2/profiles*


*See: /cygdrive/d/DATA/Puppet/1_Projekte/hochguertel.biz/Entwicklung/#_Modules/roles_and_profiles*


*See: http://www.craigdunn.org/2012/05/239/*


*See: http://sysadvent.blogspot.de/2012/12/day-13-configuration-management-as-legos.html*


*See: http://danielvigueras.com/node-definition-in-hiera-with-puppet-roles-and-profiles/*


*See: https://puppetlabs.com/presentations/designing-puppet-rolesprofiles-pattern*


	git subtree add --prefix modules/roles git@gitlab.hochguertel.biz:puppet-modules/hochguertel-roles.git master --squash
	git push origin production
	
	git subtree add --prefix modules/profiles git@gitlab.hochguertel.biz:puppet-modules/hochguertel-profiles.git master --squash
	git push origin production



	git subtree pull --prefix modules/profiles git@gitlab.hochguertel.biz:puppet-modules/hochguertel-profiles.git master --squash
	git subtree pull --prefix modules/roles git@gitlab.hochguertel.biz:puppet-modules/hochguertel-roles.git master --squash
	git push origin production

	
	git subtree add --prefix modules/nginx https://github.com/jfryman/puppet-nginx.git master --squash
	git push origin production

	git subtree add --prefix modules/gcc git@github.com:puppetlabs/puppetlabs-gcc.git master --squash
	git push origin production
	

*Contribute Back:*

	git subtree push --prefix modules/profiles git@gitlab.hochguertel.biz:puppet-modules/hochguertel-profiles.git `date +%d_%m_%Y`


*Contribute Back: Module: gitlab*

	git subtree pull --prefix modules/gitlab https://github.com/tobiashochguertel/puppet-gitlab.git master --squash

	git subtree push --prefix modules/gitlab https://github.com/tobiashochguertel/puppet-gitlab.git  `date +%d_%m_%Y`

	
	git subtree add --prefix modules/redis https://github.com/thomasvandoren/puppet-redis.git master --squash

	git push origin production
	

	# Issue: Contributes everything back which was changed!
	# shoudl only commit back what is expeceted - like for thes new feature (branch).

	#-> Forking and working with your fork-master branch. Merged the pull request there. Let the branches existing for later integration into base-fork master.
	
	git subtree pull --prefix modules/gitlab git@github.com:tobiashochguertel/puppet-gitlab.git master --squash

	git push origin production

**MCollective:**

*Overview:*
1. (Middleware Service)
2. (Mcollective Client)
3. (Mcollective Server)

*(Middleware Service):*

	git subtree add --prefix modules/mcollective https://github.com/puppet-community/puppetlabs-mcollective.git master --squash
	git push origin production

	git subtree add --prefix modules/datacat git@github.com:richardc/puppet-datacat.git master --squash
	git push origin production


	git subtree add --prefix modules/java_ks git@github.com:puppetlabs/puppetlabs-java_ks.git master --squash
	git push origin production

	git subtree add --prefix modules/activemq git@github.com:puppetlabs/puppetlabs-activemq.git master --squash
	git push origin production

	git subtree add --prefix modules/rabbitmq git@github.com:puppetlabs/puppetlabs-rabbitmq.git master --squash
	git push origin production

*Site_mcollective module - Generating and Storing Certificates:*

We will be using SSL for our MCollective installation, so we will need to provide some certificates for the mcollective module. We will need to generate a cert for our admin usewr as well. ***We can do this on the Puppet CA server using the `puppet cert` command.*** Our admin's username is `hochguertelto`, so we crate a cert named `hochguertelto`.

	0 root@puppet:~ # puppet cert generate hochguertelto
	Notice: hochguertelto has a waiting certificate request
	Notice: Signed certificate request for hochguertelto
	Notice: Removing file Puppet::SSL::CertificateRequest hochguertelto at '/var/lib/puppet/ssl/ca/requests/hochguertelto.pem'
	Notice: Removing file Puppet::SSL::CertificateRequest hochguertelto at '/var/lib/puppet/ssl/certificate_requests/hochguertelto.pem'

	mkdir -p /root/site_mcollective/files/certs
	mkdir -p /root/site_mcollective/files/client_certs
	mkdir -p /root/site_mcollective/files/private_keys


	cp /var/lib/puppet/ssl/certs/hochguertelto.pem /root/site_mcollective/files/certs
	cp /var/lib/puppet/ssl/certs/puppet.hochguertel.biz.pem /root/site_mcollective/files/certs
	cp /var/lib/puppet/ssl/certs/ca.pem /root/site_mcollective/files/certs

	cp /var/lib/puppet/ssl/private_keys/hochguertelto.pem /root/site_mcollective/files/private_keys
	cp /var/lib/puppet/ssl/private_keys/puppet.hochguertel.biz.pem /root/site_mcollective/files/private_keys

	tree /root/site_mcollective/files
	
	tar -cf site_mcollective_files.tar /root/site_mcollective/files/

	echo "Kopieren der Datei 'site_mcollective_files.tar' auf den Arbeitsrechner. Dort entpacken wir diese wieder und kopieren das Verzeichniss 'files' in das puppet-modul: 'site_mcollective'."
	
	git subtree add --prefix modules/site_mcollective git@gitlab.hochguertel.biz:puppet-modules/hochguertel-site_mcollective.git master --squash
	git push origin production

	git subtree add --prefix modules/erlang git@github.com:garethr/garethr-erlang.git  master --squash
	git push origin production

	git subtree pull --prefix modules/rabbitmq git@github.com:centromere/puppetlabs-rabbitmq.git patch-1 --squash
	git push origin production
	

*Testen des RabbitMQ Services: (Middleware Service)*
	
*File: ~/.rabbitmqadmin.conf Datei mit folgendem Inhalt:*

	[localhost]
	hostname = localhost
	port = 15671
	username = hochguertelto
	password = HJkc7ih8s0EQPGIUpxjC
	ssl = True
	ssl_key_file = /etc/rabbitmq/server_private.pem
	ssl_cert_file = /etc/rabbitmq/server_cert.pem
	
	[puppet.hochguertel.biz]
	hostname = puppet.hochguertel.biz
	port = 15671
	username = hochguertelto
	password = HJkc7ih8s0EQPGIUpxjC
	ssl = True
	ssl_key_file = /etc/rabbitmq/server_private.pem
	ssl_cert_file = /etc/rabbitmq/server_cert.pem
	
	[puppet]
	hostname = puppet
	port = 15671
	username = hochguertelto
	password = HJkc7ih8s0EQPGIUpxjC
	ssl = True
	ssl_key_file = /etc/rabbitmq/server_private.pem
	ssl_cert_file = /etc/rabbitmq/server_cert.pem

*Cmd: rabbitmqadmin*

	0 root@puppet:~
	# rabbitmqadmin -N puppet.hochguertel.biz  list users
	+---------------+------------------------------+---------------+
	|     name      |        password_hash         |     tags      |
	+---------------+------------------------------+---------------+
	| guest         | Ti1NTsK0NOtYVSnQQQha0/haxzs= | administrator |
	| hochguertelto | bghAgoj9acdT/bfCjmyT0Oer144= | administrator |
	| mcollective   | MqAHJJC3eKhbISNoRTF74dL3dY4= |               |
	+---------------+------------------------------+---------------+

*(Mcollective Client):*






**Hiera:**

	git@gitlab.coldsleep.hochguertel.biz:puppet-projekte/hochguertel-biz-infrastructure-hiera.git
	
Siehe: -> **2) Renaming branch "master" to "production"**
und:
	
![Q:\Schnellspeicherort\2015-04-06 20_20_37-Greenshot.png](http://i.imgur.com/b71PF9k.png)
![Q:\Schnellspeicherort\2015-04-06 20_21_09-Greenshot.png](http://i.imgur.com/QdErdoK.png)
![Q:\Schnellspeicherort\2015-04-06 20_21_31-Greenshot.png](http://i.imgur.com/FX4MuI7.png)
![Q:\Schnellspeicherort\2015-04-06 20_21_57-Greenshot.png](http://i.imgur.com/Ec9Mimm.png)
![Q:\Schnellspeicherort\2015-04-06 20_22_22-Greenshot.png](http://i.imgur.com/wmRXQhf.png)
![Q:\Schnellspeicherort\2015-04-06 20_23_01-Greenshot.png](http://i.imgur.com/2ijg4Ha.png)


	git clone git@gitlab.hochguertel.biz:puppet-projekte/hochguertel-biz-infrastructure-hiera.git
	cd hochguertel-biz-infrastructure-hiera

	touch hiera.yaml
	git add hiera.yaml
	git commit -m "Adding Hiera System hiera.yaml file."
	git push
	
	



*Post-receive Hook: git push sync*

*File: /hochguertel-biz-infrastructure/modules/profiles/manifests/scm_service_git.pp*

	  @@gitlab::custom_hooks::post_receive { 'puppet-projekte_hochguertel-biz-infrastructure-hiera':
	    ensure                              => present,
	    namespace                           => "puppet-projekte",
	    project                             => "hochguertel-biz-infrastructure-hiera",
	    gitlab_branch                       => '7-9-stable',
	    gitlabshell_branch                  => 'v2.6.0',
	    template_vars                       => {
	      repository_url                      => "git@gitlab.hochguertel.biz:puppet-projekte/hochguertel-biz-infrastructure-hiera.git",
	      deployment_path                     => "/etc/puppet/hiera",
	      deployment_login_privatekey_abspath => "/home/git/.ssh/id_rsa",
	      deployment_login_user               => "puppet",
	      deployment_login_ip                 => "puppet.hochguertel.biz",
	    },
	    template_file                       => "gitlab/custom_hooks/post_receive.erb",
	    tag                                 => "${::domain}",
	  }


**deployment_path                     => "/etc/puppet/hiera",**

	mkdir /etc/puppet/hiera
	chown puppet:puppet -R /etc/puppet/hiera
	
	rm /etc/hiera.yaml
	ln -s /etc/puppet/hiera/production/hiera.yaml /etc/puppet/hiera.yaml
	ln -s /etc/puppet/hiera.yaml /etc/hiera.yaml
	mkdir /etc/puppet/data/
	chown puppet:puppet -R /etc/puppet/data

	echo "The above commands to create and install the Hiera Configuration files..."
	echo "... are now manged by puppet-module: standards."

*Note:*
The `datadir` has changed!
New `datadir` is now inside of the git-repository of `project-name` with prefix `-hiera`. Which is updated via the post-receive hook of the `-hiera` repository by the usage of the `git-sync` script.

	:yaml:
		:datadir:	/etc/puppet/hiera/production/data

*Hiera Commandline Utility:*

	echo "---" >> /etc/puppet/hiera/data/global.yaml
	echo "puppetmaster: 'puppet.hochguertel.biz' >> /etc/puppet/hiera/data/defaults.yaml
	
	hiera puppetmaster

![Q:\Schnellspeicherort\2015-04-06 21_49_10-Greenshot.png](http://i.imgur.com/IypOvWR.png)

	echo "---" >> /etc/puppet/hiera/data/defaults.yaml
	echo "puppetmaster: 'puppet.hochguertel.biz'" >> /etc/puppet/hiera/data/defaults.yaml
	
	hiera -d puppetmaster

![Q:\Schnellspeicherort\2015-04-06 21_52_13-Greenshot.png](http://i.imgur.com/yOJNg0Z.png)
![Q:\Schnellspeicherort\2015-04-06 21_56_20-Greenshot.png](http://i.imgur.com/zbmNBmh.png)

*Note:*

- Hiera default behavior is to return the result of the first successfully lookup!
	- "defaults.yaml" before "global.yaml"

**Hiera with dynamic Environment:**

Hiera `datadir`: `/etc/puppet/hiera/production/data`

*File: hiera.yaml (Auschnitt)*

	:hierarchy:
	  - defaults
	  - "%{clientcert}"
	  - "%{environment}"
	  - global

`- "%{environment}"` = means that the Puppet variable `$environment` will be used as a part of the hierarchy.

*Environment/Branch: production*
*File: /etc/puppet/hiera/production/data/production.yaml*
	
	---
	syslog_server: 'loghost.pro-puppet.com'

*Environment/Branch: development*
*File: /etc/puppet/hiera/production/data/development.yaml*

	---
	syslog_server: 'badlogs.pro-puppet.com'


*Testing: hiera with dynamic Enviornments:*

	hiera -d syslog_server environment=production

![Q:\Schnellspeicherort\2015-04-06 22_20_52-Greenshot.png](http://i.imgur.com/KBqveuO.png)

*Current Hiera Configuration File:*
Validator: [YAML Lint](http://www.yamllint.com/ "YAML Lint")

	---
	:backends:
	 - yaml
	:hierarchy:
	 - defaults
	 - environment/%{environment}/data/fqdn/%{fqdn}
	 - environment/%{environment}/data/osfamily/%{osfamily}/%{lsbdistcodename}
	 - environment/%{environment}/data/osfamily/%{osfamily}/%{lsbmajdistrelease}
	 - environment/%{environment}/data/osfamily/%{osfamily}/%{architecture}
	 - environment/%{environment}/data/osfamily/%{osfamily}/common
	 - environment/%{environment}/data/modules/%{cname}
	 - environment/%{environment}/data/modules/%{caller_module_name}
	 - environment/%{environment}/data/modules/%{module_name}
	 - environment/%{environment}/data/common
	 - "%{clientcert}"
	 - "%{environment}"
	 - global
	
	:yaml:
	# datadir is empty here, so hiera uses its defaults:
	# - /var/lib/hiera on *nix
	# - %CommonAppData%\PuppetLabs\hiera\var on Windows
	# When specifying a datadir, make sure the directory exists.
	  :datadir:	/etc/puppet/hiera/production/data

*Hiera Configuration for Complex Data Structures:*

- arbitrarily data `(willkürlich / belibig)`
- Arrays
- Hashes
- Composite data structures

*Returning Structured Data*

	---
	ntp_servers:
	 - clock1
	 - clock2
	 - clock3
	testuser:
	 gid: 1001
	 uid: 1001
	 gecos: Puppet Test User
	 shell: /bin/bash

*Array Merging*

Hiera data's structures (hierarchy) can also be merged. This means we can configure Hiera to return an array for a key with elements of the array coming from different levels of the hierarchy.

![Q:\Schnellspeicherort\2015-04-06 23_15_06-Greenshot.png](http://i.imgur.com/S0Gzgfy.png)

*merge_behavior: native* = https://docs.puppetlabs.com/hiera/1/lookup_types.html#native-merging


*File: /etc/puppet/hiera.yaml*
 
	---
	:backends:
	 - yaml
	:hierarchy:
	 - defaults
	 - environment/%{environment}/data/fqdn/%{fqdn}
	 - environment/%{environment}/data/osfamily/%{osfamily}/%{lsbdistcodename}
	 - environment/%{environment}/data/osfamily/%{osfamily}/%{lsbmajdistrelease}
	 - environment/%{environment}/data/osfamily/%{osfamily}/%{architecture}
	 - environment/%{environment}/data/osfamily/%{osfamily}/common
	 - environment/%{environment}/data/modules/%{cname}
	 - environment/%{environment}/data/modules/%{caller_module_name}
	 - environment/%{environment}/data/modules/%{module_name}
	 - environment/%{environment}/data/common
	 - "%{clientcert}"
	 - "%{environment}"
	 - global
	
	:yaml:
	# datadir is empty here, so hiera uses its defaults:
	# - /var/lib/hiera on *nix
	# - %CommonAppData%\PuppetLabs\hiera\var on Windows
	# When specifying a datadir, make sure the directory exists.
	  :datadir:	/etc/puppet/hiera/production/data
	
	:merge_behavior: native
 

*merge_behavior: deep* = https://docs.puppetlabs.com/hiera/1/lookup_types.html#deep-merging-in-hiera--120

Merge Behavior `deep` needs the RubyGem: `deep_merge`.

![Q:\Schnellspeicherort\2015-04-06 23_20_45-Greenshot.png](http://i.imgur.com/3WzXhOL.png)
![Q:\Schnellspeicherort\2015-04-06 23_29_09-Greenshot.png](http://i.imgur.com/gW1ziJU.png)

	:merge_behavior: deeper

*merge_behavior: deeper* = https://docs.puppetlabs.com/hiera/1/lookup_types.html#example

[Better Explaination of Hiera Merge Behavior!](https://ask.puppetlabs.com/question/13592/when-to-use-hiera-hiera_array-and-hiera_hash/ "Better Explaination of Hiera Merge Behavior!")

[Hiera Function Overview](https://docs.puppetlabs.com/references/latest/function.html#hiera "Hiera Function Overview")

[Hiera-2](https://docs.puppetlabs.com/hiera/2.0/index.html "Hiera-2")

[Github:startrek | This module demonstrates the use of data-in-modules from Puppet 3.3.0+](https://github.com/pro-puppet/puppet-module-startrek "Github:startrek | This module demonstrates the use of data-in-modules from Puppet 3.3.0+")


**Hiera an Data in Modules**

![Q:\Schnellspeicherort\2015-04-08 11_50_44-Greenshot.png](http://i.imgur.com/RJcKNUx.png)
![Q:\Schnellspeicherort\2015-04-08 11_51_19-Greenshot.png](http://i.imgur.com/3GpDoMy.png)

	git subtree add --prefix modules/module_data git@github.com:ripienaar/puppet-module-data.git master --squash


![Q:\Schnellspeicherort\2015-04-08 11_52_21-Greenshot.png](http://i.imgur.com/43dEg9b.png)
![Q:\Schnellspeicherort\2015-04-08 11_52_55-Greenshot.png](http://i.imgur.com/UNrOxYh.png)

