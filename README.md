Chef-Repo
=========

I just recently started with Chef few weeks back and coming from Puppet background i had quite a good amount of Chef experience and learning curve to share.

**Sections ...**

  - Install Chef Server (Open Source)
  - Install Chef Client
  - Setup Admin User and Knife (Chef Workstation)
  - Chef Environments
  - Chef Roles
  - Chef Data Bags
  - Chef Cookbooks / Recipes
  - Git Version Controlled - Chef Repositories 
  - Rebuild a Chef Server
  - How does my Setup looks like
  - References


**Kickstart ..**

It took me some time to understand Chef architecture and start writing cookbooks with multiple environments and couple of application roles.

Coming from puppet background hiera was the first puzzle to sort out in Chef world. It is important to spend some time on understanding Chef Environment, Role and Data Bag, before starting with Cookbooks. 

Version Controlling was the only thing missing. chef-repo gave a good start point to split out Chef resources into different git repositories. There are number of ways you can manage Chef Server via Git, the approach followed in this document is described in section "Git Version Controlled - Chef Repositories". 


This could be a tutorial for *newbie's* like *me* to start working with Chef from Installation to Managing different Environments.

> This document is an individual approach of managing 
> Infrastructure with Chef (Open Source). 
> For Chef complete tutorial and guidelines 
> follow [Chef Wiki] and [Chef Docs]


Install Chef Server (Open Source)
-----------

Chef Installation is as easy as it can get. Download the Chef server package or follow bash install method by following [chef install].

**Customize Chef Server Details Before Setup**

This is something i have yet to explore on next Chef server build.

Details like the Chef server name, using wild card or multiple server alias certificate, domain name etc. are to setup according to infrastructure before Chef server setup. It could be possible to change it after the server setup but it could would affect existing clients.


**Running on Amazon**

If you are running on amazon platform, you might be prone to runlist error. It may have been fixed by now in latest releases in which case just ignore below section otherwise update file '/opt/chef-server/embedded/cookbooks/runit/recipes/default.rb' as a workaround.

```sh
# diff -u /opt/chef-server/embedded/cookbooks/runit/recipes/default.rb.old /opt/chef-server/embedded/cookbooks/runit/recipes/default.rb

--- /opt/chef-server/embedded/cookbooks/runit/recipes/default.rb.orig    2014-01-26 20:00:26.123165937 +0000
+++ /opt/chef-server/embedded/cookbooks/runit/recipes/default.rb.new    2014-01-26 20:03:22.030175179 +0000
@@ -29,6 +29,8 @@
   end
 when "xenserver"
   include_recipe "runit::upstart"
+when "amazon"
+  include_recipe "runit::upstart"
 else
   include_recipe "runit::sysvinit"
 end
```
Browse https://chef_server_dns and change the default admin password. Create admin client / user for different users to setup Chef Workstation.

For more information read [chef server config].

Install Chef Client
--------------

Just like server, download Chef Client package or follow typical method of bash install by following - [chef install].

Once Chef client package is installed, setup below files to bootstrap Chef client.

**Chef Client Directories**

```sh
# Create Chef Client Directories
#
mkdir /etc/chef
mkdir /etc/chef/ohai
mkdir /etc/chef/ohai/plugins
mkdir /etc/chef/ohai/hints
```

```sh
# File: /etc/chef/chef-validator.pem
# Copy the content of chef-validator.pem file
# content from server. Once chef-client 
# gets registered with Chef server this file is
# no longer required and must be removed.
```

**Chef Client Config file**

```sh
# File: /etc/chef/client.rb
# Create Chef Client Config file, below is 
# the minimal configuration to kick start 
# the client. You can modify it according
# to your requirement


# Server Configuration
chef_server_url  "https://{chef server url}/"
environment "{environment name}"
validation_client_name "chef-validator"
validation_key "/etc/chef-server/chef-validator.pem"


# Client Configuration
node_name "{chef client fqdn}"
client_key "/etc/chef/client.pem"
client_registration_retries 3
json_attribs "/etc/chef/client-config.json"

# Timeout
splay 30
interval 1800
rest_timeout 300

# Options
diff_disabled true
enable_reporting false
enable_selinux_file_permission_fixup false
file_atomic_update true
user nil
group nil

# Logging
verbose_logging false
log_level :info
log_location STDOUT

Ohai::Config[:plugin_path] << '/etc/chef/ohai/plugins'

```

For more information read [client config].

**Node Custom Attributes or run_list**

Node *run_list* or *client attributes* can be defined in "/etc/chef/client-config.json" file. 

Note that these variables will override any other variable defined in Cookbooks or Role, useful for first boot.

Ideally use knife or Chef Console to assign roles/recipes to a node.

```sh
# cat /etc/chef/client-config.json

{
  "run_list": [
    "role[common_role]",
    "role[application_role]",
    "recipe[some_recipe]",
    "recipe[some_recipe]",
    "..etc"
    ],
    "application": "webserver"
}
```

You can also define node attributes via Ohai by creating your own Ohai Plugins. 

Ohai plugins are ruby code where you can put your logic to create custom node attributes according to your environment.

e.g. File "/etc/chef/client.rb" has an entry:
"Ohai::Config[:plugin_path] << '/etc/chef/ohai/plugins'" 

which tell Ohai to include plugins from location '/etc/chef/ohai/plugins' on Chef client run.

```sh
# cat /etc/chef/ohai/plugins/application.rb
provides "application"
application "webserver"
```

If you are running on Amazon platform, ec2 hint is must be present to load Ohai ec2 metadata.

```sh
echo '{}' > /etc/chef/ohai/hints/ec2.json
```

For more info read [ohai].

For Ohai plugins [ohai custom].



Setup Admin User and Knife (Chef Workstation)
---

Knife is a Chef Management Utility. It is a set of Chef API calls and plugins to manage Chef Server. Knife can manage Chef roles, environments, data bags, cookbooks, run list, nodes and clients etc. There are some other powerful utilities available to perform same work like Berkshelf etc.

To setup a Knife Client or Workstation, user require client or user private key with admin privilege. It can be created from  Chef Console by following web document [manage auth].


For more information on Knife refer to [knife].

**Knife Setup**

Create directory '~/.chef' and create knife.rb file.

```sh
mkdir ~/.chef
touch ~/.chef/knife.rb
```

```sh
# cat ~/.chef/knife.rb

log_level                :info
log_location             STDOUT
node_name                'foo'
client_key               '~/.chef/foo.pem'
validation_client_name   'foo'
validation_key           '~/.chef/foo.pem'
chef_server_url          'https://<chef server url>'
syntax_check_cache_path  '~/.chef/syntax_check_cache'
cookbook_path            '~/chef-repo/cookbooks'
environment_path         '~/chef-repo/environments'
data_bag_path            '~/chef-repo/data_bags'
role_path                '~/chef-repo/chef_roles'
cookbook_copyright       'foo'
cookbook_license         'apachev2'
cookbook_email           'foo@foo.com'
```

For more knife.rb options [knife config].

**Test knife client**

```sh
# knife client list
chef-validator
chef-webui
webserver
...
```

Now you have a Chef Workstation from where you can create/delete/modify Chef resources. Different team members can have their own client/user id to manage different chef resources.


Chef Environments
---

Environment concept in Chef is very simple:

- environment is a simple json file or a chef dsl .rb file
- useful to define default or override attributes global to roles or specific to an attribute
- primary use of an environment is to define cookbooks version, Chef can have more than one version of cookbooks, you can map a cookbook version to an environment 
- chef default environment is _default. Unless an environment is declared in /etc/chef/client.rb or in Chef Console for a node, node by default is configured with _default environment.
- you can create 'n' numbers of environments

For more information on environment read [essentials environments].

If you have multiple environments like production, stage or qa, it is always better to bootstrap the node with the respective environment instead of _default environment.

**How to Create an Environment**

Creating an environment using knife:

```sh
# knife environment create production

{
  "name": "production",
  "description": "Production Environment",
  "cookbook_versions": {
    "cookbook_name": "= 0.1.0"
  },
  "json_class": "Chef::Environment",
  "chef_type": "environment",
  "default_attributes": {
    "some_default_variable": {}
  },
  "override_attributes": {
    "some_variable_to_override": {}
  }
}
```

This will open up editor with default attributes. For git version controlled or to edit it manually store the content into environment.json file. Later you can modify it locally without using knife or use it to create more environments. 

Better approach is to store environment.json file into a git repo. 

Whenever making any changes to environment.json file, commit it to git prior to Chef upload using knife.

```sh
knife environment from file environment.json
```

**Where to Use Environment JSON attributes**

Environment is not an easy place to define attributes. It will get out of hand before you know it.

Environment is useful to define cookbooks version. You can have multipe cookbook versions uploaded to Chef Server. You may choose which cookbook version you want to run in an environment.

In my view Environment JSON file must be kept simple. 

If there are some attributes that differs between environments and common across environment, declare them in environment.json file.

If there are some attributes common across environments, create a common role with the attributes and add to other application roles.

If you have different roles with some common attributes and you want to override some of them, use override attribute in environment.json.

When look at the attributes, its just the precedence of default & override attributes that matters with the hierarchy of attribute value. But it could be very complex and easy at the same time if not defined at the right place, something i will have to share in some time.

**Where Not to Use Environment JSON attributes**

It is very hard to put it in simple words without understanding the requirement of hierarchy flow and cookbooks design. 

Environment.json file is a very good place to make global changes to  override attributes or to define default attributes. This is something varies from one setup to another.

**Environment with a Common Role**

A Common role has simplified managing cookbooks, roles and environments.

Lets say that we have create a common role which will be attached to every other role a node will get associated with.

Any attribute you want to declare across environments, roles or cookbooks can be defined here, e.g. common packages, services, attributes etc., we can declare them in a common role. e.g. role[system]

Chef Roles
---

Chef Role is a place to store the attributes for a specific application/role and to define run_list for different environments.

- is a simple json file or a chef dsl .rb file
- a role defined run list for different environment. run_list can be list of other roles or recipes
- it stores attributes required for recipes
- you can have difference run_list for different environment, run_list is default run_list for _default environment.

For more information on [essentials roles].

```sh
# cat jump.json

{
  "name": "jump",
  "description": "Jump / SSH Login Gateway Server",
  "json_class": "Chef::Role",
  "default_attributes": {
    "groups": {
      "dev": {}
    },
    "packages": {
      "debian": {},
      "rhel": {}
    },
    "services": {
      "rhel": {},
      "debian": {}
    }
  },
  "override_attributes": {},
  "chef_type": "role",
  "run_list": [],
  "env_run_lists": {
    "production": [
      "recipe[sshd]",
      "role[system]"
    ],
    "development": [
      "recipe[sshd]",
      "role[system]",
      "role[dev_recipe]"
    ]
  }
}
```

**Common Role - role[system] for all Environments/Nodes**

role[system] is a role i have created and including to every another role or run_list of a node. 

It is a default role which has all the minimum packages, services, users, file or any other resource which must goes to every node regardless of node environment.

Just adding role[system] to run_list of other roles we will have our common base setup across the environments nodes.

**Create a Role**

To create a role first time use knife and store the role to a json file as a reference to create more roles or to make changes to a role. 

```sh
knife role create <role name>
```
This way we can put the roles json files into git repo. Like environments json file we can use git to have version controlled on roles json files.

Once a role json file is stored in a git repo, make the changes to role json file and upload to Chef server using knife:

```sh
knife role from file role.json
```

Chef Data Bags
---

Chef Data Bags are JSON Data Stores globally available for cookbooks. Data Bag is an immutable data store, primarily useful to store static data information. It also supports encryption to store data bag json file in encrypted format.

Data bag can be created as a chef dsl .rb file or a .json file.

For more information read out [essentials data bags].


Chef Cookbooks / Recipes
---

Cookbook is collection of recipes, attributes, templates etc. Cookbook is like Chef Ruby module to declare resources.
 
**Create Data Bag**

A Data Bag or Data Bag Item can be created usin knife:

```sh
knife data bag create <data bag name>
knife data bag create <data bag name> <data bag item>
```

Create a json file from first data bag item and store it in git. Later it can be used to create an item and upload directly to Chef server.

I found it very convenient to just create a data bag first using knife and later create json file manually or replciate-modify an existing one.

```sh
knife data bag from file <data bag name> <data bag item>.json
```

Now the data bags are directories in git repo with different data bag item json files in it.

Git Version Controlled - Chef Repositories
---

Version Control is quite a different approach between Puppet and Chef. From my short experience Chef server can only be managed via tools like knife via api calls or via Console.

To version Control different Chef resources we can version Control the resources itself.

I found it very useful to have different repositories for each Chef resource. Like:

- chef_data_bags - This repository has all Chef Data Bags and Items files
- chef_roles - This repository consists all the Chef roles json or .rb files
- chef_environments - This repository consists all Chef environments json or .rb files
- chef_cookbooks - This repository is categorized into three sub categories. Thanks to [Leknarf] post which made it easy for me to manage the cookbooks structure.

    - vendor_cookbooks - Store all public site downloaded cookbooks here
    - public_cookbooks - Store In-house written cookbooks here which can be shared via github or opscode or public website, they are like vendor cookbooks but developed in-house
    - private_cookbooks - Store all in-house cookbooks here which can not be shared with public
- chef_distribution - Stores any static files or small binaries files here

Now it looks very easy to maintain Git access depending on environment/files using [git acl].

**Create a New Role**

Checkout chef_roles repository:
```sh
git clone git@GIT_SERVER:chef_roles
cd chef_roles
ls
    jenkins.json    jump.json system.json
```
    
Create New role file using knife or simply copy an existing role:
```sh
cp jenkins.json new_role.json
```

Modify the new role id, description, attributes etc.
```sh
# cat new_role.json
{
  "name": "new_role",
  "description": "New Role Description",
  "json_class": "Chef::Role",
  "default_attributes": {},
  "override_attributes": {},
  "chef_type": "role",
  "run_list": [],
  "env_run_lists": {
    "production": [
    ],
    "development": [
    ]
  }
}
```

Once created, it is always better to validate JSON syntax using a json parser. This could also be set to default in git pre-hook.

Push the new role to git
```sh
git add new_roles.json
git commit -m "added new_role"
git push
```

Upload new_role to Chef using knife
```sh
knife role from file new_role.json
```

Now, you can test new role on a node. 

In future if someone else need to make a change to new_role, simply checkout chef_roles, make the change, git push and knife upload.


Rebuild a Chef Server
---

Rebuilding a Chef Server steps are similar to steps in "Install Chef Server" section. But the difference is we do not need to re-create all the cookbooks, roles, environments or data bags from scratch. 

We just need to upload all the resources we have in git repositories to new Chef Server.

There are few components which needs to backup regularly like Chef Server SSL Certificate bundle, Client Signed SSL Certificate Bundles etc. For this purpose one can use individual scripts to take backup of individual components or simple run backup/snapshot of whole Chef Server disk useful for disaster recovery.

There could be other things still remains un-explored for rebuilding a chef server, but it is a good start point to begin with.


How does my Setup looks like?
---

I will be uploading cookbooks to github chef-repo or individually if can be used standalone shortly.

If you are interested for in my Chef Practice for managing environment, roles, data bags and cookbooks, check out chef-repo.


References:
---

https://wiki.opscode.com/

http://docs.opscode.com/

http://www.slideshare.net/opscode/

http://leknarf.net/blog/2013/04/22/staying-sane-while-writing-chef-cookbooks/#.Us5JZ_a9aT4


License
----

Apache v2.0


**Open Source!**

[chef install]:http://www.getchef.com/chef/install/
[chef server config]:http://docs.opscode.com/config_rb_chef_server.html
[essentials data bags]:http://docs.opscode.com/essentials_data_bags.html
[essentials roles]:http://docs.opscode.com/essentials_roles.html
[essentials environments]:http://docs.opscode.com/essentials_environments.html
[knife config]:http://docs.opscode.com/config_rb_knife.html
[knife]:http://docs.opscode.com/knife.html
[manage auth]:http://docs.opscode.com/chef/manage_server_open_source.html
[client config]:http://docs.opscode.com/config_rb_client.html.
[ohai]:http://docs.opscode.com/ohai.html
[ohai custom]:http://docs.opscode.com/ohai_custom.html
[Leknarf]:http://leknarf.net/blog/2013/04/22/staying-sane-while-writing-chef-cookbooks/#.UujOp_a6Zcx
[git acl]:http://www.linuxforu.com/2011/01/gitolite-specify-complex-access-controls-git-server/
[Chef Wiki]:https://wiki.opscode.com/
[Chef Docs]:http://docs.opscode.com/

