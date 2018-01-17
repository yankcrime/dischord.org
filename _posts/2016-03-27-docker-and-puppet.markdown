---
layout: post
title: Docker and Puppet
date: 2016-03-27 13:06
published: true
categories:

---

> ___The principles behind this post still stand, but there's an updated workflow and tooling which solves a few of the problems mentioned below, [described here](http://dischord.org/2016/09/10/more-on-docker-and-puppet/).___

On and off for the last few weeks I've been trying to create a reasonably sane workflow that introduces [Puppet](http://puppetlabs.com) - my favourite configuration management tool - to [Docker](http://docker.com) - everyone's favourite containerization (is that a word?) technology.  I can't say I've really cracked it, I've come up with something functional but it's clear at this point that there's still a long way for existing configuration management tools - and container-based technologies - to go before we've anything reasonably coherent.  In fact, the answer might be something different altogether, but who knows.

This all started because I wanted to review the way in which we deploy some of our OpenStack services that would potentially ease the pain of upgrades when it comes to running multiple services on the same host and isolating resources such as shared python libraries.  This is a mostly solved problem for some operators (i.e using Python virtualenvs), but I fancied trying to do something cleaner and which at the same time would earn us additional geek cred by being able to tout that fact that "yes, we do run Docker" ;)

I also wanted to re-deploy the services used on my personal domain (i.e the webserver and database that sit behind this very site), so that was a good starting point.

## Opinions

There's a few ways to crack this nut, some of which I agree with and a lot of which I don't.  Here's my thoughts in no particular order:

* Containers should be ephemeral.  If you have to change something in a running container, you should be deploying from a fresh image that contains the necessary change;
* A corollary to that is running SSH within a container is a big no-no.  You shouldn't be SSH'ing into them in order to make any configuration amendments;
* Likewise, having Puppet run in a container in a master-agent setup is wrong;
* Building images for containers using shell scripts is all kinds of wrong and feels like a terrible regression and makes me very sad indeed.

There's a lot of evolving dogma around usage of containers in general - the move towards Unikernels for example - that probably fly in the face of most of what I'm doing here.  But if this approach solves a problem and scratches that itch for now then it's good enough.  With that in mind...

## Why Puppet?

Bundling shellscripts and makefiles with your Dockerfile is the wrong way to go, in my opinion.  Shell scripts are often fragile, highly opinionated in nature (as in, hard to standardise given the disparity in convention), and this results in horror shows such as inlining configuration or using sed and awk to alter configuration stanzas and variables.  I mean, look at the madness that I came up with [here](http://dischord.org/2013/08/13/docker-and-owncloud-part-2/)!

These are problems that configuration management and tools like Puppet sought to - and mostly succeeded in - solving.  On top of that, I haven't had to closely examine an application configuration file in quite some time.  Puppet becomes the configuration API - all you have to do is know how to write Puppet manifests and you can configure just about anything.  No more getting to grips with macro preprocessors just to generate your MTA's configuration!

I don't see why any of that should go away with the advent and proliferation of container-based technologies.  You still have to generate infrastructure and application configuration _somewhere_, so why not use the right tool for the job?  Puppet is one of the right tools (along with Ansible, Chef, Salt, and so on - just pick one).

## Having a go

Plenty people have attempted to this already, and in fact it's already entirely possible that others have come to the same conclusion and I've managed to miss this.  Apologies if so!  A year or so ago James Turnbull came up with [this workflow](https://puppetlabs.com/blog/building-puppet-based-applications-inside-docker) as a suggestion, but that falls pretty far from the mark when you're developing a Puppet manifest to generate your image because running [librarian-puppet](https://github.com/voxpupuli/librarian-puppet) each time is slow and adds a significant amount of time to the Docker image build process.

Instead I've settled on the following process:

* Manage modules (still using librarian-puppet for now) from the 'host' OS, the host in this case being wherever you're building your image;
* Generate a 'base' image which contains Puppet and its various dependancies, along with a couple of standard hooks so that your Hiera data and manifests are included by default when you inherit that image;
* Run `puppet apply` as part of your target image generation process;
* Clean up after yourself.

It's not perfect, but it works.

## Structure

Everything I'm about to describe is [in this GitHub repo](https://github.com/yankcrime/docker-puppet), but the salient bits of this workflow are structured as follows:

```
.
├── Dockerfile
├── Puppetfile
├── default.pp
├── docker
│   ├── cachier
│   │   └── Dockerfile
│   └── dischord
│       ├── database
│       └── webserver
├── hiera.yaml
├── modules
│   └── profile, nginx, etc.
└── hieradata
    ├── common.yaml
    ├── container
    │   ├── dischord_database.yaml
    │   └── dischord_webserver.yaml
    ├── nodes
    └── role
        ├── cachier.yaml
        ├── database.yaml
        └── webserver.yaml

```

The [top-level Dockerfile](https://github.com/yankcrime/docker-puppet/blob/master/Dockerfile) defines my base image, which includes everything needed to bootstrap Puppet as well as the additional hooks needed to get my modules, Hiera data, and manifests in place to do the configuration.  Building this base image is simply a case of running `$ docker build -t puppet.`  Pretty standard stuff.  There's some `ONBUILD` options in there to ensure that any image derived from this base include those file in the mix, which we'll then need to use `puppet apply`.

## Docker

I then have a sub-directory called `docker` which contains per-site (for now) Dockerfiles, so in my personal example these are grouped under `dischord`.  These are also straightforward, so the webserver one looks like:

```
FROM puppet:latest
MAINTAINER Nick Jones "nick@dischord.org"

ENV FACTER_role='webserver'
ENV FACTER_container='dischord_webserver'

RUN puppet apply --verbose \
		--modulepath /puppet/modules \
		--hiera_config /puppet/hiera.yaml \
		--manifestdir /puppet/ /puppet/default.pp

RUN apt-get -y clean && rm -rf /puppet

EXPOSE 80 443

CMD ["/usr/bin/supervisord", "-n"]
```

Other Dockerfiles follow almost this exact same pattern.  The key differences between generated images are a couple of top-level facts, in this case `role` and `container`, and also what's exposed service-wise.  I'm not totally settled on this arrangement but it works for me now;  I have a generic set of configuration options I want to apply to any webserver and then some container (i.e purpose) specific options that get inherited.

## Puppet

The necessary configuration data in Puppet (and Hiera) is also pretty straightforward.  Basically the top-level role defines which profile classes to include, and these then take care of including any application-specific facts plus 'workarounds'.  There's some basic configuration in place for Hiera (defining the hierarchy, duh) and then we run `puppet apply` as detailed in the above Dockerfile.  Here's the `nginx` profile class:

```puppet
class profile::nginx {

  include ::nginx

  create_resources(nginx::resource::vhost, hiera('vhosts'))
  create_resources(nginx::resource::location, hiera('locations'))

  file { '/var/www':
    ensure => 'directory',
    owner  => 'root',
    group  => 'root',
  }

  file_line { 'nginx_foreground':
    path    => '/etc/nginx/nginx.conf',
    line    => 'daemon off;',
    require => Class['::nginx'],
  }

}
```

Which then goes on to inherit configuration data from the module level and eventually the container level, so what's inherited for my website is this:

```yaml
vhosts:
  'dischord.org':
    'www_root': '/srv/www/'
    'try_files':
      - '$uri'
      - '$uri/'
      - '/index.html'
      - '/index.php?$query_string'
locations:
  'php-fastcgi':
    'vhost': 'dischord.org'
    'location': '~ \.php$'
    'fastcgi': 'unix:/var/run/php5-fpm.sock'
```

What this means is that each time I want update the configuration for my website's nginx Docker container, I just need to add a few lines to a YAML file and then trigger the build and deployment of a new container from this updated image.  Easy.  A video tells a lot of words, so here's me defining building a database image.  The first run I just build the image with the configuration as-is, and then I make a couple of changes to Hiera to define a new database and then re-generate the image:

<center><script type="text/javascript" src="https://asciinema.org/a/40506.js" id="asciicast-40506" async></script></center>

Using Puppet to manage configuration in this way doesn't add much in the way of build time, in fact we're mostly waiting for packages to download (albeit via a local cache) and install.

## Problems

There's a few things that suck about this workflow and using Puppet to build Docker images:

* Docker hasn't been designed with this sort of configuration management in mind, so having to copy the entirety of that repo into each and every container at build time feels bad.  It'd be nice if you could just mount the Puppet-related stuff at build time to save you having to clean up after yourself, and indeed other people have come up with different use-cases for this option but so far it's not been implemented;
* It breaks the image layers philosophy (see above point).  You've got two technologies competing for their idea of idempotency, and because of the way these two interact Puppet eventually wins.  The user loses because ultimately it takes longer to generate an image;
* Idiosyncracies to do with how containers act differently from a standard Linux installation, i.e init, that cause problems with some modules.  In my case I've ended up standardising on [supervisord](http://supervisord.org) to handle running applications in containers, but sometimes you have consider workaround for modules that assume a working Upstart configuration, i.e the [puppetlabs-mysql](https://forge.puppetlabs.com/puppetlabs/mysql) module which errors with:

```
Debug: Executing '/sbin/initctl --version'
Error: /Stage[main]/Mysql::Server::Service/Service[mysqld]: Could not evaluate: undefined method `[]' for nil:NilClass
```

To fix this, you need to override the default service provider to be plain ol' `init`, i.e in the case of the `puppetlabs-mysql` module:

```
mysql::server::service_provider: 'init'
```

* Puppet's lack of support for the more 'minimal' Linux distributions leads to slightly 'bloated' images.  Taking advantage of the convenience of Puppet's fantastic community and ecosystem means you really need to stick to Ubuntu or RHEL, which in turn means fatter images.  There's ongoing work for better support for things like Alpine Linux but there's a way to go yet.  For me, the convenience outweighs the extra bandwidth requirements.


## Next steps

There's a still a ways to go before this workflow becomes useful.  Missing steps from this post are getting a private Docker Registry stood up, and then deploying containers from our generated images automatically.  Again, [Puppet can be used here](https://forge.puppetlabs.com/garethr/docker) but that's for another post at a later date.

It'd be even better if we made more use of service discovery tools such as Consul to handle some aspects of configuration but really that's outside the scope of this simplified example.  Of course, it could just be that over the coming few weeks and months I bin this stuff off altogether but for now it's a fun diversion, if nothing else ;)
