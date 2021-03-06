Puppet Gluster
==============

Table of Contents
-----------------

1. [Overview](#overview)
2. [Custom Facts](#custom-facts)
3. [Classes](#classes)
4. [Resources](#resources)
5. [Examples](#examples)
6. [Contributing](#contributing)
7. [Copyright](#copyright)

Overview
--------

This module installs and configures servers to participate in a [Gluster](http://www.gluster.org/) Trusted Storage Pool, create or modify one or more Gluster volumes, and mount Gluster volumes.

Also provided with this module are a number of custom Gluster-related facts.

Custom Facts
------------

* `gluster_binary`: the full pathname of the Gluster CLI command
* `gluster_peer_count`: the number of peers to which this server is connected in the pool.
* `gluster_peer_list`: a comma-separated list of peer hostnames
* `gluster_volume_list`: a comma-separated list of volumes being served by this server
* `gluster_volume_#{vol}_bricks`: a comma-separated list of bricks in each volume being served by this server
* `gluster_volume_#{vol}_options`: a comma-separared list of options enabled on each volume
* `gluster_volume_#{vol}_ports`: a comma-separated list of ports used by the bricks in the specified volume.

The `gluster_binary` fact will look for an [external fact](http://docs.puppetlabs.com/guides/custom_facts.html#external-facts) named `gluster_custom_binary`. If this fact is defined, `gluster_binary` will use that value. Otherwise the path will be searched until the gluster command is found.

Classes
-------

### params.pp ###
This class establishes a number of default values used by the other classes.

You should not need to include or reference this class directly.

### repo.pp ###
This class optionally enables the upstream [Gluster.org](http://download.gluster.org/pub/) repositories.  

Currently, only the yum repo type is implemented.

    class { ::gluster::repo:
      version => '3.5.2',
    }

### install.pp ###
This class handles the installation of the Gluster packages (both server and client).

If the upstream Gluster repo is enabled, this class will install packages from there. Otherwise it will attempt to use native OS packages.

Currently only RHEL 6 and RHEL 7 provide native Gluster packages.

    class { gluster::install:
      server  => true,
      client  => true,
      repo    => true,
      version => 3.5,
    }

### client.pp ###
This class installs **only** the Gluster client package(s). If you need to install both the server and client, please use the `install.pp` (or `init.pp`) classes.

    class { ::gluster::client:
      repo    => true,
      version => '3.5.2',
    }

Use of `gluster::client` is not supported with either `gluster::install` or `gluster::init`.

### service.pp ###
This class manages the `glusterd` service.

    class { ::gluster::service:
      ensure => running,
    }

### init.pp ###
This class implements a basic Gluster server.

In the default configuration, this class exports a `gluster::peer` defined type for itself, and then collects any other exported `gluster::peer` resources for the same pool for instantiation.

This default configuration makes it easy to implement a Gluster storage pool by simply assigning the `gluster` class to your Gluster servers: they'll each export their `gluster::peer` resources, and then instantiate the other servers' `gluster::peer` resources.  

The use of exported resources assume you're using PuppetDB, or some other backing mechanism to support exported resources.

If a `volumes` parameter is passed, the defined Gluster volume(s) can be created at the same time as the storage pool. See the volume defined type below for more details.

    class { ::gluster:
      repo    => true,
      client  => false,
      pool    => 'production',
      version => '3.5',
      volumes => {
        'data1' => {
          replica => 2,
          bricks  => [ 'srv1.local:/export/brick1/brick',
                       'srv2.local:/export/brick1/brick',
                       'srv1.local:/export/brick2/brick',
                       'srv2.local:/export/brick2/brick', ],
          options => [ 'server.allow-insecure: on',
                       'nfs.disable: true', ],
                   }
                 }
    }

Resources
-------------
 
### gluster::peer ###
This defined type creates a Gluster peering relationship.  The name of the resource should be the fully-qualified domain name of a peer to which to connect. An optional `pool` parameter permits you to configure different storage pools built from different hosts.

With the exported resource implementation in `init.pp`, the first server to be defined in the pool will find no peers, and therefore not do anything.  The second server to execute this module will collect the first server's exported resource and initiate the `gluster peer probe`, thus creating the storage pool.

Note that the server being probed does not perform any DNS resolution on the server doing the probing. This means that the probed server will report only the IP address of the probing server.  The next time the probed client runs this module, it will execute a `gluster peer probe` against the originally-probing server, thereby updating its list of peers to use the FQDN of the other server.

See [this mailing list post](http://supercolony.gluster.org/pipermail/gluster-users/2013-December/015365.html) for more information.

    gluster::peer { 'srv1.domain:
      pool => 'production',
    }

### gluster::volume ###
This defined type creates a Gluster volume. You can specify a stripe count, a replica count, the transport type, a list of bricks to use, and an optional set of volume options to enforce.

Note that creating brick filesystems is up to you. May I recommend the [Puppet Labs LVM module](https://forge.puppetlabs.com/puppetlabs/lvm) ?

When creating a new volume, this defined type will ensure that all of the servers hosting bricks in the volume are members of the storage pool. In this way, you can define the volume at the time you create servers, and once the last peer joins the pool the volume will be created.

Any volume options defined will be applied after the volume is created but before the volume is started.

In the event that the list of volume options active on a volume does not match the list of options passed to this defined type, no options will be removed by default. You must set the `$remove_options` parameter to `true` in order for this defined type to remove options.

Note that adding or removing options does not (currently) restart the volume.

    gluster::volume { 'data1':
      replica => 2,
      bricks  => [
                   'srv1.local:/export/brick1/brick',
                   'srv2.local:/export/brick1/brick',
                   'srv1.local:/export/brick2/brick',
                   'srv2.local:/export/brick2/brick',
                 ],
      options => [
                   'server.allow-insecure: on',
                   'nfs.ports-insecure: on',
                 ],
    }

### gluster::volume::option ###
This defined type applies [Gluster options](https://github.com/gluster/glusterfs/blob/master/doc/admin-guide/en-US/markdown/admin_managing_volumes.md#tuning-options) to a volume.

In order to ensure uniqueness across multiple volumes, the title of each `gluster::volume::option` must include the name of the volume to which it applies.  The format for these titles is `volume:option_name`:

    gluster::volume::option{ 'g0:nfs.disable':
      value => 'on',
    }
 
To remove an option, set the `ensure` parameter to `absent`:

    gluster::volume::option{ 'g0:server.allow-insecure':
      ensure => absent,
    }

### gluster::mount ###
This defined type mounts a Gluster volume.  Most of the parameters to this defined type match either the gluster FUSE options or the [Puppet mount](http://docs.puppetlabs.com/references/3.4.stable/type.html#mount) options.

    gluster::mount { '/gluster/data1':
      ensure    => present,
      volume    => 'srv1.local:/data1',
      transport => 'tcp',
      atboot    => true,
      dump      => 0,
      pass      => 0,
    }

Examples
--------
Please see the examples directory.

Contributing
------------

Pull requests are warmly welcomed!

Copyright
---------

Copyright 2014 [CoverMyMeds](https://www.covermymeds.com/) and released under the terms of the [MIT License](http://opensource.org/licenses/MIT).
