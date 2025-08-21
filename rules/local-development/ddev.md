# Working with DDEV

This project is very likely to be set up with ddev. Check by running `ddev describe`. All ddev instructions only need to be follow if ddev describe indicates the project is set up with ddev.

### Path Mapping
The host's project root is mapped to <containerPath>/var/www/html</containerPath> inside the main (web) container. File changes are reflected both ways.

### Commands
Running commands like npm,yarn and composer have to be executed inside of ddevs container like this:

 ```bash
 # Execute `phpcs` composer command in a wordpress plugin environment.
 ddev exec --dir=/var/www/html/wp-content/plugins/my-plugin composer phpcs
 
 # List the web container’s docroot contents
 ddev exec ls /var/www/html
 ```

### Documentation
* `ddev` command usage: https://ddev.readthedocs.io/en/stable/users/usage/cli/
* 
