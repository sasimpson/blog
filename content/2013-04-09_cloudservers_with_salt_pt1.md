Date: 2013-08-07
Title: CloudServers with Salt
Author: Scott
Category: rackspace

[Salt](http://docs.saltstack.com) is a python based infrastructure management
 tool.  With it you can use a master to control a bunch of minions.  One feature
 I like is the ability to spin up servers in the cloud.  To start with you'll need
 salt and salt-cloud installed. On your master you have to create a file with
 all your image profiles in /etc/salt/cloud.profiles, mine looks like:

    ubuntu_12_04_512:
        provider: openstack
        size: 512MB Standard Instance
        image: Ubuntu 12.04 LTS (Precise Pangolin)

    ubuntu_11_04_512:
        provider: openstack
        size: 512MB Standard Instance
        image: Ubuntu 11.04    

 Here are my provider settings I use to spin up a server on Rackspace
  CloudServers (/etc/salt/cloud):

    minion:
        master: myhost.mydomain.com

    provider: rackspace

    OPENSTACK.identity_url: 'https://identity.api.rackspacecloud.com/v2.0/tokens'
    OPENSTACK.compute_name: cloudServersOpenStack
    OPENSTACK.compute_region: region
    OPENSTACK.tenant: tenant_id
    OPENSTACK.user: userid
    OPENSTACK.apikey: yourkey
    OPENSTACK.protocol: ipv4

    log_file: /var/log/salt/cloud

This will create a CloudServer:

    salt-cloud -p openstack_512 testweb1

That's it!  To destroy it:

    salt-cloud -d testweb1

Salt uses _States_ to control how you interact with the minions.  I can have 
Apache and MySQL installed with a few configurations.  Salt uses yaml syntax for 
the configuration files.  Usually they are stored in /srv/salt/ and the base file
 is called top.sls.  My top.sls looks like:

    base:
      testweb1:
        - www.apache
        - db.mysql

the __testweb1__ part tells which host to run the next states on.  each state 
is in a list below the host.  I have two listed, one is to get apache setup and
 one is to get mysql setup.  Salt knows where to look, the dotted structure is 
 for directories and files.  In the _www_ directory, I've got an _apache.sls_ file
 which has the settings for setting up apache.  That looks like (/srv/salt/www/apache.sls):

    apache2:
      pkg:
        - installed
      service:
        - running

    php5:
      pkg:
        - installed

When this runs, it tells the package management system on the installed OS to
 install Apache2 and PHP5.  Pretty sweet.  The MySQL part looks like (/srv/salt/db/mysql.sls):

    mysql-server:
      pkg:
        - installed
      service:
        - running

now you should have mysql and apache with php up and running.  
