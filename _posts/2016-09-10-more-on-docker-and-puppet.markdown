---
layout: post
title: More on Docker and Puppet
date: 2016-09-10
published: true
categories:
---

I gave a presentation at the inaugural - and awesome - [Docker Manchester](http://www.meetup.com/Docker-Manchester/events/232283804/) meetup a few weeks back which detailed my ["building Docker images with Puppet"](http://dischord.org/2016/03/27/docker-and-puppet/) workflow and why we've headed down that path.  Since then it's been refined a fair bit and so my original post is now a bit out of date.  So, time for an update!

Headline changes are:

* No longer building a 'base' Puppet image;
* A switch to Puppet 4 and the [AIO](https://puppet.com/blog/say-hello-to-open-source-puppet-4) packaging which includes [r10k](https://github.com/puppetlabs/r10k);
* Use of the latter for handling installation of modules on a per-image basis;
* Discovery of ['Rocker'](https://github.com/grammarly/rocker) which makes mounting volumes at image build time (more on that shortly) possible, as well as an ability to template the build process.

Most of this was prompted and inspired by the brilliant work that [Gareth Rushgrove](http://www.morethanseven.net/) is doing on pretty much exactly this problem.

Here's the skinny from a working example that we use at [DataCentred](http://www.datacentred.co.uk).  It's basically what I discussed during my presentation - the steps required to build an image from which we can deploy a container that runs [OpenStack's Horizon](http://docs.openstack.org/developer/horizon/).

> _I've also updated my [personal repo](https://github.com/yankcrime/docker-puppet) to follow this approach, so have a poke around there for something to clone and mess around with._

## Overview

The tl;dr summary is that this approach uses Puppet in a mode which runs once during the image build process and applies any relevant configuration.  It uses Rocker to share data volumes across builds and to template common aspects of configuration and build artefacts.


## Introducing Rocker 

Our starting point is this Dockerfile which containers a few Rocker-specific options:

```
FROM ubuntu:16.04

ENV DOCKER_BUILD_DOMAIN='sal01.datacentred.co.uk'
ENV FACTER_domain=${DOCKER_BUILD_DOMAIN:-vagrant.test} FACTER_role='horizon'
ENV PUPPET_AGENT_VERSION="1.6.1" UBUNTU_CODENAME="xenial"
ENV PATH=/opt/puppetlabs/server/bin:/opt/puppetlabs/puppet/bin:/opt/puppetlabs/bin:$PATH

MOUNT /opt/puppetlabs /etc/puppetlabs /root/.gem
MOUNT ./puppet/hieradata:/hieradata
MOUNT ~/.config/keys:/keys

RUN apt-get update && \
    apt-get install -y lsb-release git wget && \
    wget https://apt.puppetlabs.com/puppetlabs-release-pc1-"$UBUNTU_CODENAME".deb && \
    dpkg -i puppetlabs-release-pc1-"$UBUNTU_CODENAME".deb && \
    rm puppetlabs-release-pc1-"$UBUNTU_CODENAME".deb && \
    apt-get update && \
    apt-get install --no-install-recommends -y puppet-agent="$PUPPET_AGENT_VERSION"-1"$UBUNTU_CODENAME" && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

RUN /opt/puppetlabs/puppet/bin/gem install hiera-eyaml deep_merge r10k:2.2.2 --no-ri --no-rdoc

COPY horizon/Puppetfile /
COPY puppet/modules/profile /profile
COPY puppet/default.pp /
COPY Rockerfile /Dockerfile

RUN r10k puppetfile install --moduledir /etc/puppetlabs/code/modules && \
    ln -s /profile /etc/puppetlabs/code/modules/profile && \
    puppet apply /default.pp --verbose --show_diff  --summarize --hiera_config=/hieradata/hiera.yaml && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

EXPOSE 80

CMD ["/usr/bin/supervisord", "-n"]

TAG horizon:mitaka
```

Amongst other things, [Rocker](https://github.com/grammarly/rocker) provides a `MOUNT` directive which you can use to share volumes across image builds as well as being able mount in local directories at build time.  This is great because it now means it's possible to have per-image module installation in a way that doesn't slow the process down too much.  It also solves a couple of the other flaws I pointed out in my previous post, chief amongst which is the concern that you're baking in a crapton of extra guff - configuration data, Puppet modules - that you simply don't need.  Using Rocker means it's mounted only when it's needed - i.e at build time.

Maintaining these on a per-image basis is a pain in the butt as you end up with a slew of Dockerfiles that are almost exactly the same, bar a few options.  Fortunately it's also possible to template your Dockerfiles with this same tool, and as all we basically need to change across builds are these handful of parameters - specifically the 'role' that's used to dicate to Puppet which configuration classes to include, which ports to expose, and what image tags to apply - this means we can keep duplication of our build artefacts down to a reasonable minimum.

If we go ahead and do that we can get this down to a couple of files that are common to all images: a templated 'Rockerfile' and a bit of YAML with all the keys / values:

```
{% raw %}
$ cat common/Rockerfile
FROM {{ .BASE }}
MAINTAINER {{ .MAINTAINER }}

ENV DOCKER_BUILD_DOMAIN={{ .DOCKER_BUILD_DOMAIN }}
ENV FACTER_role={{ .ROLE }} FACTER_domain=${DOCKER_BUILD_DOMAIN:-vagrant.test}
ENV PUPPET_AGENT_VERSION={{ .PUPPET_AGENT_VERSION }}
ENV UBUNTU_CODENAME={{ .UBUNTU_CODENAME }}
ENV PATH={{ .PATH }}

MOUNT {{ .MOUNT.puppet }}
MOUNT {{ .MOUNT.hiera }}
MOUNT {{ .MOUNT.keys }}

RUN {{ .RUN.puppet_install }}
RUN {{ .RUN.install_gems }}

COPY puppet/r10k/{{ .ROLE }} /Puppetfile
COPY {{ .COPY.profile }}
COPY {{ .COPY.defaultpp }}

RUN  {{ .RUN.puppet_apply }}

CMD {{ .CMD }}

EXPOSE {{ .EXPOSE }}
TAG {{ .TAG }}
{% endraw %}
```

And then the YAML data that's used to populate this before it's sent to the Docker build API:

```
$ cat common/common.yaml
BASE: 'ubuntu:16.04'

DOCKER_BUILD_DOMAIN: 'sal01.datacentred.co.uk'
PUPPET_AGENT_VERSION: '1.6.1'
UBUNTU_CODENAME: 'xenial'
PATH: '/opt/puppetlabs/server/bin:/opt/puppetlabs/puppet/bin:/opt/puppetlabs/bin:$PATH'
MOUNT:
  puppet: '/opt/puppetlabs /etc/puppetlabs /root/.gem'
  hiera: './puppet/hieradata:/hieradata'
  keys: '~/.config/keys:/keys'

RUN:
  puppet_install: |
    apt-get update && \
    apt-get install -y lsb-release inetutils-ping vim git wget && \
    wget https://apt.puppetlabs.com/puppetlabs-release-pc1-"$UBUNTU_CODENAME".deb && \
    dpkg -i puppetlabs-release-pc1-"$UBUNTU_CODENAME".deb && \
    rm puppetlabs-release-pc1-"$UBUNTU_CODENAME".deb && \
    apt-get update && \
    apt-get install --no-install-recommends -y puppet-agent="$PUPPET_AGENT_VERSION"-1"$UBUNTU_CODENAME" && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
  install_gems: '/opt/puppetlabs/puppet/bin/gem install hiera-eyaml deep_merge r10k:2.2.2 --no-ri --no-rdoc'
  puppet_apply: |
    r10k puppetfile install --moduledir /etc/puppetlabs/code/modules && \
    ln -s /profile /etc/puppetlabs/code/modules/profile && \
    puppet apply /default.pp --verbose --show_diff  --summarize --hiera_config=/hieradata/hiera.yaml && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

COPY:
  profile: 'puppet/modules/profile /profile'
  defaultpp: 'puppet/default.pp /'

CMD: '["/usr/bin/supervisord", "-n"]'
```


## Puppet
All Puppet-related configuration data lives under its own subdirectory.  This includes Hiera data common to all images and the usual smattering of profile classes, so the directory structure ends up something like this:

```
$ tree -d -L 2
.
├── common
└── puppet
    ├── hieradata
    ├── modules
    └── r10k
```

In order to handle Puppet module installation, per-image `r10k` Puppetfiles reside in the r10k subdirectory with a filename that reflects the rolename, i.e `puppet/r10k/horizon`.  As per before, there's still a common `default.pp` that simply contains (excluding some PATH setting):

```puppet
hiera_include('classes')

Class['apt::update'] -> Package <| |>

create_resources(supervisord::program, hiera('service'))
```

The role `ENV` variable - but this time passed in via the `rocker build` command line - is responsible for defining what classes are included as an image is created.  Our Hiera hierarchy has a role subdirectory with files named in accordance with this role variable.  So for this example, `puppet/hiera/role/horizon.yaml` contains:

```puppet
---
classes:
  - '::profile::openstack::horizon'

service:
  'horizon':
    'command': '/usr/sbin/apachectl -DFOREGROUND'
    'stdout_logfile': '/dev/stdout'
    'stderr_logfile': '/dev/stderr'
    'stdout_logfile_maxbytes': '0'
    'stderr_logfile_maxbytes': '0'

branding::horizon::release: 'mitaka'
```

And then the `::profile::openstack::horizon` class is just a couple of includes:

```puppet
class profile::openstack::horizon {
  include ::horizon
  include ::branding::horizon

  file { '/var/log/apache2/horizon_access.log':
    target  => '/dev/stdout',
    require => Package['httpd'],
  }

  file { [ '/var/log/apache2/horizon_error.log', '/var/log/apache2/error.log' ]:
    target  => '/dev/stderr',
    require => Package['httpd'],
  }

}
```

The `file` resources are to make sure everything logs to either `STDOUT` or `STDERR` so that we can leverage Docker's logging capabilities.  The rest of the configuration data is mostly in Hiera, scoped to module or role.

## Build Process

With all that in place, kicking off a build with Rocker is simple enough:

```bash
rocker build -f common/Rockerfile --vars common/common.yaml \
--var EXPOSE="80" --var TAG=horizon:mitaka --var ROLE=horizon .
```

Apart from the Puppet and r10k configuration data, all we have to do to build a different image is slightly amend the command line options which specify the role, which ports to expose, and the tag.  To kick off a [Glance](http://docs.openstack.org/developer/glance/) image build for example, it's simply a case of amending a few of those `--var` parameters:

```bash
rocker build -f common/Rockerfile --vars common/common.yaml \
--var EXPOSE="9191 9292" --var TAG=glance:mitaka --var ROLE=glance .
```

Protip:  `rocker` has a `-print` option that you can use to make sure the resulting Rockerfile is what you'd expect without actually triggering a build.

Resulting images are then pushed to a private registry and then we're using Puppet again to deploy containers from these images.  Good times!

