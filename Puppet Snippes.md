# Puppet Domain Specific Language (DSL) Snippets

- [Style Guide](https://docs.puppetlabs.com/guides/style_guide.html)
- [Puppet Best Practice2](http://projects.puppetlabs.com/projects/puppet/wiki/puppet_best_practice2) 

**Switch for Operation Systems:**

*File: puppetlabs-mysql/manifests/params.pp*

	class mysql::params {
	  case $::osfamily {
	    'RedHat': {
	      if $::operatingsystem == 'Fedora' and (is_integer($::operatingsystemrelease) and $::operatingsystemrelease >= 19 or $::operatingsystemrelease == "Rawhide") {
	        $client_package_name = 'mariadb'
	        $server_package_name = 'mariadb-server'
	      } else {
	        $client_package_name = 'mysql'
	        $server_package_name = 'mysql-server'
	      }
	      $basedir             = '/usr'
	      $config_file         = '/etc/my.cnf'
	      $datadir             = '/var/lib/mysql'
	      $log_error           = '/var/log/mysqld.log'
	      $pidfile             = '/var/run/mysqld/mysqld.pid'
	      $root_group          = 'root'
	    }
	
	    'Debian': {
	      ## More parameters defined here
	    }
	  }
	}

*File: puppetlabs-mysql/manifests/server.pp*

	class mysql::server (
	  $config_file             = $mysql::params::config_file,
	  $manage_config_file      = $mysql::params::manage_config_file,
	  $old_root_password       = $mysql::params::old_root_password,
	  $override_options        = {},
	  $package_ensure          = $mysql::params::server_package_ensure,
	  $package_name            = $mysql::params::server_package_name,
	  $purge_conf_dir          = $mysql::params::purge_conf_dir,
	  $remove_default_accounts = false,
	  $restart                 = $mysql::params::restart,
	  $root_group              = $mysql::params::root_group,
	  $root_password           = $mysql::params::root_password,
	  $service_enabled         = $mysql::params::server_service_enabled,
	  $service_manage          = $mysql::params::server_service_manage,
	  $service_name            = $mysql::params::server_service_name,
	  $service_provider        = $mysql::params::server_service_provider,
	  # Deprecated parameters
	  $enabled                 = undef,
	  $manage_service          = undef
	) inherits mysql::params {
	
	  ## Puppet goodness goes here
	}

**Hiera defaults pattern:**

*File: puppetlabs-mysql/manifests/server.pp*

	class mysql::server (
	  $config_file             = hiera('mysql::params::config_file', 'default value'),
	  $manage_config_file      = hiera('mysql::params::manage_config_file', 'default value'),
	  $old_root_password       = hiera('mysql::params::old_root_password', 'default value'),
	  ## Repeat the above pattern
	) {
	
	  ## Puppet goodness goes here
	}

**Hybrid data model:**

*File: puppetlabs-mysql/manifests/server.pp*
	
	class mysql::server (
	  $config_file             = hiera('mysql::params::config_file', $mysql::params::config_file),
	  $manage_config_file      = hiera('mysql::params::manage_config_file', $mysql::params::manage_config_file),
	  $old_root_password       = hiera('mysql::params::old_root_password', $mysql::params::old_root_password),
	  ## Repeat the above pattern
	) inherits mysql::params {
	
	  ## Puppet goodness goes here
	}



**Defined Types:**
- [Language: Defined Resource Types](https://docs.puppetlabs.com/puppet/latest/reference/lang_defined_types.html "https://docs.puppetlabs.com/puppet/latest/reference/lang_defined_types.html")

*File: [/etc/puppetlabs/puppet/modules/apache/manifests/vhost.pp](https://github.com/puppetlabs/puppetlabs-apache/blob/master/manifests/vhost.pp)*

	# Define

    define apache::vhost (
		$port,
		$docroot,
		$servername = $title,
		$vhost_name = '*'
	) {
      include apache # contains Package['httpd'] and Service['httpd']
      include apache::params # contains common config settings
      $vhost_dir = $apache::params::vhost_dir
      file { "${vhost_dir}/${servername}.conf":
        content => template('apache/vhost-default.conf.erb'),
          # This template can access all of the parameters and variables from above.
        owner   => 'www',
        group   => 'www',
        mode    => '644',
        require => Package['httpd'],
        notify  => Service['httpd'],
      }
    }


	# Call
	
	apache::vhost { "example.com":
	  docroot  => '/var/www/html',
	  priority => '25',
	}



**Roles:**

*File: modules/role/manifests/uat_server.pp*

	  # Example role
	  
	  class role::uat_server {
	    include profiles::base
	    include profiles::customapp
	    include profiles::test_tools
	  }


**Profiles:**

*File: modules/profiles/manifests/gitlab.pp*

	# Example profile:

	class profiles::gitlab {

	   ## Hiera lookups
	   # - not yet
	   # see: http://garylarizza.com/blog/2014/02/17/puppet-workflow-part-2/
	   # $site_name = hiera('profiles::gitlab::site_name')
	   
	   ## Create user
	   
	   ## Configure mysql
	   
	   ## Configure apache
	   
	   ## Configure gitlab
	  
	}

**Templates:**
- https://docs.puppetlabs.com/guides/templating.html


*File: modules/gitlab/templates/custom_hooks/post-receive.erb*

	# This file is managed by Puppet. DO NOT EDIT.
	# Getting a List of All Variables
	<% scope.to_hash.keys.each do |k| -%>
	<%= k %>
	<% end -%>


**Header for Puppet Managed Files:**

	# This file is managed by Puppet. DO NOT EDIT.
