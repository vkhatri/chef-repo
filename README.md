Chef-Repo
=========

I has been some time i started working on Opscode Chef. Coming from Puppet background it has been a good experience and learning curve. I think by putting this document together, my little experience could be useful to a newbie like me to start Managing infrastructure using Opscode Chef OSS.


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
  - References


**Kickstart ..**



If you are new to Configuration Management System, check out below links on Opscode Chef:

> https://wiki.opscode.com/display/chef/Home

> http://docs.opscode.com/

> http://docs.opscode.com/chef_overview.html


Chef documentation is very vast and some times it gets very tricky to choose a best way of doing things. Chef supports pure Ruby implementation, which means you can write your own code within Chef DSL.

There are multiple ways of doing a single thing, you can simply put everything in a cookbook using simple ruby program or you could take the benefits of roles/environments/data bags etc to write a cookbook. And if you want to get serious about creating a standard cookbook, check out [Custom LWRP] or [Custom Ruby LWRP]. 

Its very easy to build Chef resources without following proper standards or enough thinking, i have had this experience in past and it is not easy to go back & set things right. Before you know, it gets out of hands and you have no choice but keep going with the things as they are.

Laying out Chef Environment, Role and Data Bag data attributes properly is very essential to write a proper cookbook in such a way that future addon or resource addition will be easy & could be handled with minimal effort of changing cookbook. It might not be true to every scenario but it sure does help to cover requirements like handling packages management etc. 

If you are coming from puppet background, hiera is the first puzzle to sort out in Chef world. You can achieve it at many levels and many ways. 

Version Controlling is another important piece when it comes to any Configuration Management System. It is what decides how fast system can recover from disaster or reverted back to last known good state. chef-repo gave a good start point towards version control. Splitting out Chef resources into different git repositories provides even more granular control. There are number of ways you can manage Chef Server via Git, the approach followed in this document is described in section "Git Version Controlled - Chef Repositories". 


> This document is an individual approach of managing 
> Infrastructure using Opscode Chef (Open Source). 
> For Chef complete documentation and reference 
> check out [Chef Wiki] and [Chef Docs].


Install Chef Server (Open Source)
-----------

Chef Installation is very simple, just download the Chef server package or follow bash install method by following [chef install].

```sh
e.g. On RHEL bsed platform
# Download Package
  wget https://opscode-omnibus-packages.s3.amazonaws.com/el/6/x86_64/chef-11.8.2-1.el6.x86_64.rpm 

# Install Chef Server Package
  rpm -ivh chef-11.8.2-1.el6.x86_64.rpm

# Create/Modify Chef Server configuration file as per requirement
  vim /etc/chef-server/chef-server.rb

# Configure Chef Server
  chef-server-ctl reconfigure
```

**How to Change Chef Server FQDN or SSL Certificate**

To apply internal SSL Certificates or to Change SSL Certificate / Chef Server FQDN, first modify `/etc/chef-server/chef-server.rb`:

```sh
  /etc/chef-server/chef-server.rb

api_fqdn "CHEF_SERVER_FQDN"
nginx['server_name']        = "CHEF_SERVER_FQDN"
nginx['ssl_certificate']        = "/etc/chef-server/CHEF_SERVER_FQDN.crt"
nginx['ssl_certificate_key']    = "/etc/chef-server/CHEF_SERVER_FQDN.key"
nginx['ssl_company_name']   =   'COMPANY'
nginx['ssl_country_name']   = 'COUNTRY'
nginx['ssl_email_address']  =   'mail@mydomain.in'
nginx['ssl_locality_name']  = 'CITY'
nginx['ssl_organizational_unit_name']    = 'ORGANIZATION'
#nginx['ssl_port']   = SSL_PORT # Default 443
nginx['ssl_state_name'] = 'STATE'
```

Reconfigure Chef Server by executing:

```sh
  chef-server-ctl reconfigure
```

Do not forget to run test after making any change to Chef Server:

```sh
  chef-server-ctl test
```

**Running on Amazon**

If you are running on amazon platform, you might be prone to runlist error. It might have been fixed by now in later Chef server releases in which case just ignore below section otherwise update file '/opt/chef-server/embedded/cookbooks/runit/recipes/default.rb' as a workaround.

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
Browse https://chef_server_dns and change the default admin password. Create admin client/user for different users to setup Chef Workstation.

For more information read [chef server config].

Install Chef Client
--------------

Just like server, download Chef Client package or follow typical method of bash install by following [chef install].

Once Chef client package is installed, create below files to bootstrap Chef client.

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
# To Forward logs to a file:
# log_location '/var/log/chef/chef-client.rb'

Ohai::Config[:plugin_path] << '/etc/chef/ohai/plugins'

Dir.glob(File.join("/etc/chef", "client.d", "*.rb")).each do |conf|
  Chef::Config.from_file(conf)
end
```

For more information read [client config] or checkout [chef-client] cookbook.

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

If you are running on Amazon platform, ec2 hint must be present to load Ohai ec2 metadata.

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

Environment is useful to define cookbooks version, environment global attributes and override roles attributes. You can have multipe cookbook versions uploaded to Chef Server and can choose which version you want to run in an environment.


- environment JSON file must be kept simple, other than overriding roles attributes or global roles/cookbooks attributes try to use roles/cookbooks instead of environment

- environment.json file can have attributes differs between environment, like to test new java_version, development environment can have newer version for testing, while production still running on older version

- If environments shares list of common attributes, create a common role with those attributes and add it to each environment for other roles.

- If you have different roles with some common attributes and you want to override some of them, use environment to override.

Attribute value is just the precedence of default & override attributes spreaded acrossed cookbooks, roles and environment. But it could get complex and out of hand if not defined properly.


**Where Not to Use Environment JSON attributes**

This is something that can be learnt over time, but try to avoid environment for any attributes declaration unless have to.


**Environment with a Common Role - role[system]**

A Common role is a simplified approach to manage common attributes, run_list for cookbooks, other roles and environments.

Lets create a common role 'role[system]' to store all global attributes, recipe and run_list which can be attached to other roles.

Any attribute you want to declare across environments, roles or cookbooks can be defined in this common e.g. common packages, services, attributes, run_list etc.


Chef Roles
---

Chef Role is a collection of attributes and run_list of roles/recipes. It is more like an application type which knows what variables needs to be declared and which recipes/roles needs to run for that application.

In a role each environment can have different run_list of roles/recipes.

- role is a simple json file or chef dsl .rb file
- role is used to define environment specific run_list
- role can store attributes required for recipes

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

role[system] is a role i found very useful to use with each role or run_list of a node. 

It is a default role which has all the minimum packages, services, users, file or any other resource declaration which must be setup to every node regardless of environment.

Just adding role[system] to run_list of other role or node we will have our common base setup across the environments nodes.

**Create a Role**

To create a role you can use blow knife command:

```sh
knife role create <role name>
```
You can also create a .json or .rb file manually or copy from an existing one. By Creating role.json files, they can be stored in a git repo. Like environments json file we can use git to have version controlled on roles json files.

Once a role json file is created, use knife to upload to Chef server:

```sh
knife role from file role.json
```

Chef Data Bags
---

Chef Data Bags are immutable JSON Data Stores globally available for cookbooks. They are useful to store static data information or attributes, e.g. use/group/keys information. Data Bags supports encryption to store data bag json file in encrypted format.

For more information read out [essentials data bags].

**Create Data Bag**

A Data Bag or Data Bag Item can be created usin knife:

```sh
knife data bag create <data bag name>
knife data bag create <data bag name> <data bag item>
```

Data bag json file can also be created manually. As Data Bags are json files they can easily be maintained in git.

To upload Data Bag item directly from a json file:

```sh
knife data bag from file <data bag name> <data bag item>.json
```

Now the data bags are directories in git repo with different data bag item json files in it.


Chef Cookbooks / Recipes
---

Cookbook is collection of recipes, attributes, templates etc. Cookbook is like Chef Ruby module to declare resources.


**Create a Cookbook**

```sh
# knife cookbook create -o . sample_cookbook
** Creating cookbook sample_cookbook
** Creating README for cookbook: sample_cookbook
** Creating CHANGELOG for cookbook: sample_cookbook
** Creating metadata for cookbook: sample_cookbook
```

For detail documentation refer to [chef cookbooks].


Git Version Controlled - Chef Repositories
---

Version Control is quite a different approach between Puppet and Chef. From my short experience Chef server can only be managed via tools like knife or via Console.

We can version control Chef by version controlling Chef resources.

Instead of maintaing a single chef-repo repository, we can split chef-repo into different repositories:

- chef_data_bags - This repository has all Chef Data Bags and Items files
- chef_roles - This repository consists all the Chef roles json or .rb files
- chef_environments - This repository consists all Chef environments json or .rb files
- chef_cookbooks - This repository is categorized into three sub categories. Thanks to [Leknarf] post which made it easy to manage cookbooks structure.

    - vendor_cookbooks - Store all public site downloaded cookbooks here
    - public_cookbooks - Store In-house written cookbooks here which can be shared via github or opscode or public website, they are like vendor cookbooks but developed in-house
    - private_cookbooks - Store all in-house cookbooks here which can not be shared with public
    
- chef_distribution - Stores any static files or small binaries files here

Git access can be privileged per repository for different environment using [git acl].

**Create a New Chef Role using git**

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

And finally you can test the new role. 


Rebuild a Chef Server
---

Rebuilding a Chef Server steps are similar to steps in "Install Chef Server" section. 

But the difference in rebuilding an existing server or adding more Chef servers is that we do not need to re-create all the cookbooks, roles, environments or data bags. We can simply use git repositories to upload resources to new Chef server.

There are few components which needs to backup regularly like Chef Server SSL Certificate bundle, Client Signed SSL Certificate Bundles etc. For this purpose one can use individual scripts to take backup of individual components or simple run backup/snapshot of whole Chef Server disk useful for disaster recovery.

There could be other things still remains un-explored for rebuilding a chef server, but it is a good start point to begin with.


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
[chef cookbooks]:http://docs.opscode.com/essentials_cookbooks.html
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
[Custom LWRP]:http://docs.opscode.com/lwrp_custom_resource.html
[Custom Ruby LWRP]:http://docs.opscode.com/lwrp_custom_provider_ruby.html
[chef-client]:https://github.com/opscode-cookbooks/chef-client
