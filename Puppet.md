# Puppet: Virtual resources, Exported Resources and Stored Configuration

[TOC]

## Virtual resources

|   Types of Export Resources   |                        Description                         |
|-------------------------------|------------------------------------------------------------|
| Virtual resources             | Resources that may be required by multiple configurations. |
| Exported resources            | Take Resources from other hosts and use them on an node.   |
| Stored configuration          | mechanism to store Virtual and Exported resources.         |

### Usage: *virtual resources:*

**Steps Overview:**

1. *Define* virtual resources.
2. *Call* virtual resources.
3. *Establish a relationship* between virtual resources.

**Syntax Guide:**

1. *Define* virtual resources.

    `**@***type*`

2. *Call* virtual resources.

    **Spaceship-Operator:** 
    `*type* **<| ** *parameter-name* == *'looking-up value'* ** |>**`

    + Spaceship-Operator Syntax does not throw an error if there is no virtual resource found.
    + Spaceship-Operator Syntax use any parameter to collect resources. All collected resources will be called and created by puppet. *Like all virtual User resources which are in the same group.*

    **realize() function:**
    `**realize( ***type*[*title-value*]** )**`

    + Realsize-Function Syntax throws an error if there is no virtual resource found.

3. Establish a relationship between virtual resources.

        User <| gid == 'mysql' |> { before => Package['mysql'] }

        # 
        User <| gid == 'mysql' |> { before => Package['mysql'] }

    + A virtual resource can be extended by adding a block containing parameters.

**Working Example with Steps 1. Define, 2. Call a virtual resource and 3. Establish a relationship **

        # Define:
        @user { 'mysql':
            ensure  => present,
            uid => '27',
            gid => '27',
            home=> '/var/lib/mysql',
            shell   => '/bin/bash',
        }

        # Call: by Spaceship-Operator:
        User <| title == 'mysql'|>

        # Call: by realsize()-Function:
        realize(User['mysql'])

        # Establish a relationship: 
        User <| gid == 'mysql' |> { before => Package['mysql'] }

## Exported Resources


**Usage: *Exported resources:***


1. Define virtual resources. 
Syntax: **@@***type*

2. Call virtual resources.
Syntax: Spaceship-Operator: *type* **<<| ** *parameter-name* == *'looking-up value'* ** |>>**


	   # Define

		@@sshkey { "${::fqdn}_dsa":
		    host_aliases => [ $::fqdn, $::hostname, $::ipaddress ],
		    type => dsa,
		    key => $::sshdsakey,
		}
		
		@@sshkey { "${::fqdn}_rsa":
		    host_aliases => [ $::fqdn, $::hostname, $::ipaddress ],
		    type => rsa,
		    key => $::sshrsakey,
		    
		}
	
	   # Call

		Sshkey <<| |>> {
		  ensure => present,
		}

**The Stack:**

|    Layer    |                      Function                     |  Representation |                     |
|-------------|---------------------------------------------------|-----------------|---------------------|
| Roles:      | Business Logic                                    | *Puppet module* | <-- ENC: Classifier |
| Profiles:   | Implementation                                    | *Puppet module* | <-- Hiera: Data     |
| Components: | Resource modelling                                | *Puppet module* | <-- Hiera: Data     |
| Resources:  | State modelling of underlying OS implementations. | *Puppet module* |                     |

**Naming Conventions:**

| Layer       | Conventions                                             | Example                                       |
| :-          | :-                                                      | -:                                            |
| Components: | Should be named after what they manage.                 | apache, ssh, mysql                            |
| Profiles:   | Should be named after the logical stack they implement. | database, bastion, email                      |
| Roles:      | Should be named in business logic convention.           | uat_server, web_cluster, application, archive |

**Abstraction:**

| Type      | Puppet   |
| :-        | -:       |
| Data      | hiera    |
| Providers | types    |
| Resources | classes  |
| Classes   | modules  |
| Modules   | profiles |


## Stored Configuration




## Relationships

| Variants of Relationships |                                        Description                                         |
|---------------------------|--------------------------------------------------------------------------------------------|
|  Metaparameters           | `before`, `require`, `subscribe` and `notify`.                                             |
| Relationship-chaining     |     replace the *Metaparameters* with arrow operators.                                     |
|                           | This allows to declared relationships outside of the blocks where the resource is defined. |

### Relationship-chaining Syntax

| Chaining-Arrows | Name               | Description                                                                                                                                                                             |
| :-------------- | :---               | :----------                                                                                                                                                                             |
| `->`            | ordering arrow     | Causes the resource on the left to be applied before the resource on the right. Written with a hyphen and a greater-than sign.                                                          |
| `~>`            | notification arrow | Causes the resource on the left to be applied first, and sends a refresh event to the resource on the right if the left resource changes. Written with a tilde and a greater-than sign. |
 
### Reversed Forms
Both chaining arrows have a reversed form (`<-` and `<~`). As implied by their shape, these forms operate in reverse, causing the resource on their right to be applied before the resource on their left.

**Keep in mind for best Pratice: ** As the majority of Puppet’s syntax is written left-to-right, these reversed forms can be confusing and are not recommended.


## Further more Informations

- [https://docs.puppetlabs.com/guides/language_history.html](https://docs.puppetlabs.com/guides/language_history.html)
