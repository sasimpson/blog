Date: 2015-10-29
Title: How to Create a RethinkDB Cluster on Rackspace Carina
Author: Scott
Category: rackspace

It's been a while since I've posted anything, most of my posts have been on my [ham radio blog](http://blog.kf5way.com), so I thought I would post some cool things I've been playing with recently.

Rackspace just announced [Carina](https://getcarina.com) which is a container runtime environment based on OpenStack Magnum.  When I signed up, I wanted to have something I could accomplish with this new technology and thought of RethinkDB and its clustering capabilities.  Sure you can do all this on your desktop/laptop, but what fun is that?

To get started you've got to sign up for Carina at [https://getcarina.com](https://getcarina.com).  Carina is a runtime environment for container engines and in this case, it is using docker.  In order to get started, follow [this](https://getcarina.com/docs/getting-started/getting-started-on-carina/) handy tutorial.  Once you can create a docker swarm and the first instance, come back here.

Now, here is the fun part of how easy it is to create a RethinkDB cluster.  We have to create a master instance, using the RethinkDB docker image

    docker run -d -P --name rethinkdb-m rethinkdb

Now after you've done that you'll probably want to see what ports are exposed on which IPs:

    docker port rethinkdb-m

In my case:

    $> docker port rethinkdb-m
    28015/tcp -> 172.99.77.87:32787
    29015/tcp -> 172.99.77.87:32786
    8080/tcp -> 172.99.77.87:32788

The web console for my cluster is at **172.99.77.87:32788**

Now if you've played with RethinkDB, you've already been here a few times before.  Here is where the cool stuff starts.  

We want to now create 5 docker instances that will use the master we just created.  Docker has a command called **link** that lets you link one container to another.  In addition, the /etc/hosts file of the new containers we are going to create will have an IP entry for *rethinkdb-m*. Our creation is going to look like this:

    docker run -d -P --name rethinkdb-1 --link rethinkdb-m:masterdb rethinkdb rethinkdb --join rethinkdb-m --bind all

So what happens there?  We override the normal command of starting rethink with

    rethinkdb --join rethinkdb-m --bind all

The join tells RethinkDB to join the other cluster, on the default port using the hostname of rehinkdb-m.  If you were to create a docker instance and just load bash, you could grep /etc/hosts and there would be an entry for rethinkdb-m:

    $> docker run -i -t --link rethinkdb-m:masterdb ubuntu grep "rethinkdb-m" /etc/hosts                                                                                                              ssimpson@outback
    172.17.0.12	masterdb 3d5f6dcc5cd8 rethinkdb-m
    172.17.0.12	rethinkdb-m
    172.17.0.12	rethinkdb-m.bridge

So how do we create a bunch of containers all running RethinkDB linked back to the master:

    for i in {1..5}; do
    docker run -d -P --name rethinkdb-$i --link rethinkdb-m:masterdb rethinkdb rethinkdb --join rethinkdb-m --bind all;
    done;

And:

![wow](http://share.scottic.us/rethinkdb_cluster.png)

Very cool, big props to the Carina team at Rackspace and the RethinkDB folks for having an awesome project!
