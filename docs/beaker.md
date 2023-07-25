---
title: Setup beaker and rvm for local acceptance tests upstream modules
description: Using rvm, bundler and vagrant, we will describe how to setup and configure all those tools to be able to run all tests defined in the upstream module from syntax to acceptance.
tags:
  - vagrant
  - beaker
  - rvm
  - puppet module testing
---
# Installing rvm

For rvm, we need te latest version (not stable) due to openssl1 incompatibility
with older ruby versions.

```
gpg --keyserver keyserver.ubuntu.com --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E37D2BAF1CF37B13E2069D6956105BD0E739499BDB
curl -sSL https://get.rvm.io | bash
. /home/johan/.bash_profile
rvm list known
```

This should show at least the following available ruby versions:

```
[ruby-]2.7[.8]
[ruby-]3.0[.6]
[ruby-]3.1[.4]
[ruby-]3[.2.2]
```

Install one of the following versions or both.

```
rvm install 3.2.2
rvm install 2.7
```

When errors occur please check $HOME/.rvm/log/1689604671_ruby-2.7.2/make.log

Most errors occur due to missing *-devel pacakes.
Retry after missing packages are installed.

To select a rvm managed ruby version:

```
rvm use ruby-3.2.2
```

To set a default rvm managed ruby version:

```
rvm use ruby-3.2.2 default
```

# Preparing your upstream module

Clone the upstream module you want to change and cd into the new directory.

In this directory, first select the desired ruby version you want to use.

```
rvm use ruby-3.2.2
```

Prepare the director for local testing.

```
bundle install
```

When connected to the sfpd network, this can lead to an SSL error. One can use
in that case.

```
GEM_SOURCE=http://rubygems.org bundle install
```

Now one can run the available syntax, unit and other tests defined for the
module locally.

```
bundle exec rake help
bundle exec rake validate
bundle exec rake test
bundle exec rake spec
...
```

# Accpetance tests with beaker and vagrant

Although there are vagrant rpm's available for Fedora, thos seems not to work
properly when using rvm managed rubies.  The problem here is that many dependant
gems will not b available or not usable due to mismatch in eg. versions and
more.

So we need to install vagrant inside the rvm managed ruby of the module.

Also the latest vagrant version on https://rubygems.org is really outdated, and
hashicorp won't upload any gems anymore.  So we need to fetch the gem from there
github page, (https://github.com/hashicorp/vagrant/releases)

## Packages needed on your system

### NFS (optional)

This is only needed when sunced folders over NFS are used.  Not needed for our
acceptance tests.

Enabling NFS synced folders:

```
sudo dnf install nfs-utils && sudo systemctl enable nfs-server
$ sudo firewall-cmd --permanent --zone=libvirt --add-service=nfs3 \
    && sudo firewall-cmd --permanent --zone=libvirt --add-service=nfs \
    && sudo firewall-cmd --permanent --zone=libvirt --add-service=rpc-bind \
    && sudo firewall-cmd --permanent --zone=libvirt --add-service=mountd \
    && sudo firewall-cmd --reload
```

### libvirtd

```
sudo dnf install libvirt-devel
```

## Installling vagrant inside bundler

This needs to be done in every project/directory where one has executed the **rm
use** command.  This is needed because the ruby version is bound to that
directory and upwards only.

```
bundle exec gem install ~/Downloads/vagrant-2.3.7.gem
```

Add the following to the Gemfile:

```
group :vagrant do
  gem 'vagrant', '~> 2.3.7'              :require => false
end
```

TODO: find a lean way to add custom gems without spoiling the upstream Gemfile
(aka Gemfle.local)

Verify if everything is working:

```
[dejoh@lx-20202046 puppet-mongodb]$ bundle exec vagrant --version
Vagrant 2.3.7
[dejoh@lx-20202046 puppet-mongodb]$
```

## Installing vagrant plugins

Following plugins will be installed:

* vagrant-libvirt
* vagrant-hostmanager
* vagrant-registration
* vagrant-proxyconfig

```
bundle exec vagrant plugin install vagrant-libvirt --plugin-version 0.12.2 --plugin-source https://rubygems.org
bundle exec vagrant plugin install vagrant-hostmanager --plugin-source https://rubygems.org
bundle exec vagrant plugin install vagrant-registration --plugin-source https://rubygems.org
bundle exec vagrant plugin install vagrant-proxyconfig --plugin-source https://rubygems.org

bundle exec vagrant plugin list

vagrant-hostmanager (1.8.9, global)
vagrant-libvirt (0.12.2, global)
  - Version Constraint: 0.12.2
vagrant-proxyconf (2.0.10, global)
vagrant-registration (1.3.4, global)
```

# Configure the SUT list (beaker.yaml)

Example: Beaker.yaml used for the puppet-mongodb module.

```
HOSTS:
  redhat-7-x86_64:
    roles:
      - agent
    platform: el-7-x86_64
    box: generic/rhel7
    hypervisor: vagrant_libvirt
  redhat-8-x86_64:
    roles:
      - agent
    platform: el-8-x86_64
    box: generic/rhel8
    hypervisor: vagrant_libvirt
  redhat-9-x86_64:
    roles:
      - agent
    platform: el-9-x86_64
    box: generic/rhel9
    hypervisor: vagrant_libvirt
  centos-7-x86_64:
    roles:
      - agent
    platform: el-7-x86_64
    box: alvistack/centos-7
    hypervisor: vagrant_libvirt
  centos-8-x86_64:
    roles:
      - agent
    platform: el-8-x86_64
    box: alvistack/centos-8-stream
    hypervisor: vagrant_libvirt
  centos-9-x86_64:
    roles:
      - agent
    platform: el-9-x86_64
    box: alvistack/centos-9-stream
    hypervisor: vagrant_libvirt
  ubuntu-2304-x86_64:
    roles:
      - agent
    platform: ubuntu-2304-x86_64
    box: alvistack/ubuntu-23.04
    hypervisor: vagrant_libvirt
  ubuntu-2204-x86_64:
    roles:
      - agent
    platform: ubuntu-2204-x86_64
    box: alvistack/ubuntu-22.04
    hypervisor: vagrant_libvirt
  debian-10-x86_64:
    roles:
      - agent
    platform: debian-10-x86_64
    box: alvistack/debian-10
    hypervisor: vagrant_libvirt
  debian-11-x86_64:
    roles:
      - agent
    platform: debian-11-x86_64
    box: alvistack/debian-11
    hypervisor: vagrant_libvirt
  debian-12-x86_64:
    roles:
      - agent
    platform: debian-12-x86_64
    box: alvistack/debian-12
    hypervisor: vagrant_libvirt
  opensuse-15-x86_64:
    roles:
      - agent
    platform: opensuse-15-x86_64
    box: opensuse/Leap-15.5.x86_64
    hypervisor: vagrant_libvirt
```

## Configure generic setings for vagrant

Default settings can be set in the file $HOME/vagrant.d/Vagrantfile.
We used this to configure generic settings for redhat registration for used rhel
boxes.

```
# some defaults for all Vagrantfile's for $USER
#
Vagrant.configure('2') do |config|

  if Vagrant.has_plugin?("vagrant-hostmanager")
    config.hostmanager.enabled = true
  end

  if Vagrant.has_plugin?("vagrant-proxyconf")
    config.proxy.enabled  = true
    config.proxy.http     = "http://proxy.example.com:8080"
    config.proxy.https    = "http://proxy.example.com:8080"
    config.proxy.no_proxy = "localhost,127.0.0.1"
  end


  if Vagrant.has_plugin?('vagrant-registration')
    config.registration.manager = 'subscription_manager'
    config.registration.force = true
    config.registration.auto_attach = true
    config.registration.username = ENV['RHEL_USER']
    config.registration.password = ENV['RHEL_PASS']
  end

  config.vm.synced_folder ".", "/vagrant", disabled: true

end
```

# Configure beaker

To use beaker with vagrant-libvirt and our cusstm SUT's, we need to set some
environment variables:

* BEAKER_HYPERVISOR=vagrant_libvirt
* BEAKER_DESTROY=no|onpass  (don't destroy the VM, or only when all tests are successfull)
* BEAKER_PROVISION=no (Reuse the box, must already exists)
* BEAKER_SETFILE=../vagrant/beaker_hosts.cfg

See also https://github.com/voxpupuli/voxpupuli-acceptance
