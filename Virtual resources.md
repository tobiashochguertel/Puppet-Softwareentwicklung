<a name="virtual-resources"></a>
## Virtual resources

|   Types of Export Resources   |                        Description                         |
|-------------------------------|------------------------------------------------------------|
| Virtual resources             | Resources that may be required by multiple configurations. |
| Exported resources            | Take Resources from other hosts and use them on an node.   |
| Stored configuration          | mechanism to store Virtual and Exported resources.         |

<a name="usage-virtual-resources"></a>
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
