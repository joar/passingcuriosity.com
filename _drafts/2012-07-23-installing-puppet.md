---
title: Installing Puppet, PuppetDB and Puppet Dashboard on Ubuntu 12.04
---

This post is a complete set of instructions for installing Puppet, PuppetDB
and Puppet Dashboard on a fresh install of Ubuntu 12.04 with the package
repository provided by PuppetLabs. It includes a few extra steps which aren't
included in the official instructions -- mostly to resolve dependencies which
are missing from the packages -- but is pretty much identical to the official
instructions.

## Install the system

I installed a fresh system with Ubuntu 12.04 using the
`ubuntu-12.04-server-amd64.iso`. I selected the following roles during install:

- OpenSSH server
- PostgreSQL database

After installation I also added the `avahi-daemon` package as I'm installing
in a VM as an experiment and don't want to lookup the dynamically assigned IP
each time I boot it.


Make sure the newly installed system is up to date:

    sudo aptitude update
    sudo aptitude upgrade

## Configure the PuppetLabs package repository

    wget http://apt.puppetlabs.com/puppetlabs-release-precise.deb
    sudo dpkg -i puppetlabs-release-precise.deb
    sudo aptitude update

## Install Puppet

    sudo aptitude install puppet

## Install PuppetDB

    sudo aptitude install puppetdb

### Configure PuppetDB

http://docs.puppetlabs.com/puppetdb/0.9/connect_puppet.html#step-2-edit-config-files


## Install Puppet Master

### Configure it to use PuppetDB

    sudo aptitude install puppetdb-terminus

## Install Puppet Dashboard

### Install `puppet-dashboard` and `mysql-server`.

    sudo aptitude install puppet-dashboard
    sudo aptitude install mysql-server
    sudo aptitude install ruby-json

You'll probably need to increate the `max_allowed_packet` setting to `32M`.

    sudo $EDITOR /etc/mysql/my.cnf
    sudo service mysql restart

Don't forget to "secure" your new MySQL installation:

    sudo mysql_secure_installation

### Create the MySQL user and database

Create a MySQL user and database for Puppet Dashboard to store its data:

    mysql -uroot -p
    CREATE DATABASE dashboard CHARACTER SET utf8;
    CREATE USER 'dashboard'@'localhost' IDENTIFIED BY 'my_password';
    GRANT ALL PRIVILEGES ON dashboard.* TO 'dashboard'@'localhost';

Now configure puppet-dashboard to use it these details for the `production`
environment:

    sudo $EDITOR /etc/puppet-dashboard/database.yml

### Install the database schema

Puppet Dashboard is a Ruby on Rails app, so you'll need to use `rake` to
install the database schema.

    cd /usr/share/puppet-dashboard
    sudo rake RAILS_ENV=production db:migrate

This failed for me until I ran both of the following commands:

    sudo gem install rack --version=1.1.0
    sudo gem install rack --version=1.1.2

Ruby package version management for the win? Then the migration should work:

    cd /usr/share/puppet-dashboard
    sudo rake RAILS_ENV=production db:migrate

### Start the service

Configure the `puppet-dashboard` service, allowing it to start:

    sudo vim /etc/default/puppet-dashboard

And then start the service:

    sudo service puppet-dashboard start

This takes a few moments and spews notices and deprecation warnings onto your
console, but should start the service. Your puppet dashboard should now be
accessible on port 3000!

### Configure puppet and puppetmaster

You'll need to update the puppet configuration on the puppetmaster to store
reports in the newly installed Puppet Dashboard:

    sudo $EDITOR /etc/puppet/puppet.conf

and add the following configuration:

> # puppet.conf (on puppet master)
> [master]
>   reports = store, http
>   reporturl = http://dashboard.example.com:3000/reports/upload

You'll also need to configure Puppet agents to send reports to the master:

    # On each puppet agent machine:
    sudo $EDITOR /etc/puppet/puppet.conf

And make sure the `[agent]` section contains at least:

> # puppet.conf (on each agent)
> [agent]
>   report = true

### Enable worker processes

Enable the Puppet Dashboard worker processes:

    sudo $EDITOR /etc/default/puppet-dashboard-workers

And then start the service

    sudo service puppet-dashboard-workers start

### Switch to passenger


# Notes

Firewall

PuppetDB: accept incoming on 8081

Once installed and hooked up puppetmaster, puppetdb and puppet-dashboard:

- install ruby-json before puppet-dashboard can ask for facts.
- Need to give www-data permission to /usr/share/puppet-dashboard/certs/dashboard.private_key.pem
