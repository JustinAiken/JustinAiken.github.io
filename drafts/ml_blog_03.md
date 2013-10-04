# Using the Telephony Dev Box as a base for your own application

In my [previous blog post](http://mojolingo.com/blog/2013/using-telephony-dev-box/) I went through getting started with the [Telephony Dev Box](https://github.com/mojolingo/Telephony-Dev-Box).  This post picks up where that one left off, and moves from 'getting started' into something a bit more real world - we'll build a complete development environment for a real world voice application.

A [github repo](doesn't exist yet) has been created to host in full the results of everything covered in this post…

## Voice apps in the real world

On the adhearsion [website](http://www.adhearsion.com), the example on the front page reads:

```ruby
  def run
    answer
    resp = ask "How many woodchucks?", :limit => 1
    say "You said #{resp}. That's obviously wrong!"
  end
```

It's a cute example, but let's face it - if your application is that simple, you could probably just use Asterisk and hardcode the dialplan logic in extensions.conf - ugly, but serviceable.

#### Integration

Real applications need to be tightly integrated - with websites, SaaS's, databases, instant messaging systems, etc…  Many people coming to adhearsion have an existing Rails app they want to add telephone features to, while staying with their primary language (Ruby).

So let's get started building a development environment for this more realistic scenario.

#### Roadmap

Our goal through this post is to set up an Adhearsion application and RoR application

* The RoR application will be backed by a mySQL database
* The Adhearsion application will query the RoR app over an HTTP API
* The Adhearsion application will pass info via Redis back to the RoR app
* State information will be communicated back and forth via XMPP

You may ask why we would choose to use avenues of communication such as these instead of simply integrating ActiveRecord directly into Adhearsion?  Because it's a BAD IDEA, and we cannot recommend that approach.

Let's get started - first by stripping out unneeded things.

## That's a lot of boxes!

Although the devbox provides VM's for all supported softswitches, adhearsion itself, and various utility boxes, you shouldn't need most of those to develop a voice application.  We're going to choose a single softswitch, and base our development environment around there.

We'll use Freeswitch, so we can go through and delete everything out of the [Vagrantfile](https://github.com/mojolingo/Telephony-Dev-Box/blob/master/Vagrantfile) related to Asterisk and Prism.  We'll also get rid of the load testing tools and extra TTS engines.

#### Adhearsion - in or out?

Although you can run Adhearsion inside a VM, most of us at MojoLingo prefer to run all the Ruby apps locally on our own machine, and offload the telephone engine/services into the VM.  So we'll remove the adhearsion VM also.

Now we're left a Vagrantfile looking like this:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = 'precise64'
  config.vm.box_url = 'http://files.vagrantup.com/precise64.box'
  config.librarian_chef.cheffile_dir = "."

  config.vm.define :freeswitch do |freeswitch|
    ...
  end
end
```

#### Cleanup the Cheffile

You can also remove the cookbooks for all the apps/services just removed, leaving you a cookbook looking like this:

```ruby
#!/usr/bin/env ruby
#^syntax detection

site 'http://community.opscode.com/api/v1'

cookbook 'mojolingo-misc', path: 'mojolingo-misc'

cookbook 'apt'
cookbook 'freeswitch', git: 'https://github.com/mojolingo/freeswitch-cookbook.git', ref: 'feature/rayo'
cookbook 'chef-solo-search', git: 'https://github.com/edelight/chef-solo-search.git'
cookbook 'yum'
cookbook 'sudo'
```

## Adding services

Now our VM has gotten lonely - nothing remains but Freeswitch itself.  Let's add some recipes to flush out the environment we'll need.

#### How to add services

There are several options for provisioning your vagrant VM.  [Chef](http://docs.vagrantup.com/v2/provisioning/chef_solo.html) is a favorite around here, and what the TDB uses for it's images, but [Puppet](http://docs.vagrantup.com/v2/provisioning/puppet_apply.html) is also widely used, while [Salt](https://github.com/saltstack/salty-vagrant) and [Ansible](http://docs.vagrantup.com/v2/provisioning/ansible.html) are two newcomers that are generating lots of buzz.

You can mix and match provisioners - just because Freeswitch is provisioned with Chef doesn't mean you HAVE to use Chef for the other packages.  We'll stick with Chef for this post, but if you're going to be working on a project where there's already a staging/production environment set up, it's a great idea to use the same packages to build your dev environment that are already used.

#### Adding our services

We'll need to extend our VM with

* mysql
* redis
* ejabberd

to give us the services we chose to have our Ahn app connect to the RoR app with.

#### mysql:

We'll pick a [random mysql cookbook](https://github.com/fnichol/chef-mysql) that I've used before.  Add this line to your Cheffile:

`cookbook 'mysql', git: "https://github.com/fnichol/chef-mysql`

and set up the attributes in your Vagrantfile for a basic server:

```json
{
  {}
}
```


#### redis:

 blah blah add redis

#### ejabberd:

blah blah add ejabberd

## `vagrant up`

Finally, let's set the IP/hostname in our Vagrantbox, leaving us a final Vagrantfile looking like this:

Pick an IP near the ones used by the TDB, but different in case you need to use a stock TDB for another project, say `111.111.111.111`.
Let's pick a friendly hostname for it, say `freeswitch.local-dev.my_ahn_app.com`, and add it to your hosts file.

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = 'precise64'
  config.vm.box_url = 'http://files.vagrantup.com/precise64.box'
  config.librarian_chef.cheffile_dir = "."

  config.vm.define :freeswitch do |freeswitch|
    public_ip = "10.203.175.13"
    freeswitch.vm.hostname = "freeswitch.local-dev.mojolingo.com"

  end
end
```

`vagrant up`, and let's prepare our apps while it builds...

## Local adhearsion app + RoR app

#### Generate adhearsion app
* `ahn new testapp`

* Add the [redis](redis_gem) gem to your Gemfile..

* Add the [adhearsion-xmpp](https://github.com/adhearsion/adhearsion-xmpp) gem to your Gemfile.

* Open up the `adhearsion.rb` in the generate values, and edit the config options to point to our new VM.  (Ideally you'd set the config options with environment variables, and manage them with something like [foreman](https://github.com/ddollar/foreman), but that's a topic for another day.)

[embedded gist](gist.com/)

#### Generate RoR App
* `rails new testwebapp`

* Point your `database.yml` to the VM:

[embedded gist](gist.com/)

* Add the [blather](blather) gem to the Rails Gemfile for XMPP
* Add the same redis gem to the Gemfile
* `bundle install`
* Set up an initializer to point Redis/XMPP to the VM:

[embedded gist](gist.com/)

* Let's throw up some quick scaffolding to demonstrate how RoR can serve DB info to ahn:

`rails g scaffold Customer foo:bar`

## Connect them!

* Blah blah incoming call controller…
* Blah blah Ahn request…
* Blah blah RoR Response…
* Blah blah outgoing call controller…
* Blah blah Ahn process redis message…
* Blah blah XMPP status in RoR…
* Blah blah XMPP status in Ahn…

## Some tips
* alias `vagrant destroy && vagrant up` to something easy - you'll be using that command a lot when building a new VM! ;)
* Learn when you need to destroy and rebuild, and when `vagrant provision` is sufficient to try some changes.
* Focus on getting one service installed/configured at a time - comment out the other recipes so you don't have to wait for Freeswitch to compile to see if your mySQL recipe works.
* If you're using Virtualbox as your Vagrant host, don't start Parallels or your Mac will crash :(
* Now you have a Rails app in one tab, an Adhearsion app in another tab, and at least one tab for your VM - save all those tabs in a profile/window arrangement/tmux session! Don't waste your life new tab && `cd`ing over and over!
## Further reading / other resources
* [SOA for the Little Guys](http://www.sitepoint.com/soa-for-the-little-guys/) - With adhearsion / RoR running seperate, you're well on your way to your own little SOA…
* [Connecting Adhearsion](https://vimeo.com/55285949) - Video from 2012 Adhearsion Conference all about the different ways to wire Adhearsion up with Rails (or anything else)
* [adhearsion-xmpp](https://github.com/adhearsion/adhearsion-xmpp) - Plugin for XMPP connectivity
* [adhearsion-drb](https://github.com/adhearsion/adhearsion-drb) - Plugin for DRB connectivity
* [adhearsion-ldap](https://github.com/adhearsion/adhearsion-ldap) - Plugin for LDAP connectivity
* [Virgina](https://github.com/polysics/virginia) - Archaically-named plugin for building a 'simple, Sinatra-style web interface with your Adhearsion application'
* [Mailing List](https://groups.google.com/forum/#!forum/adhearsion) and [IRC](http://adhearsion.com/irc) - Whatever service you're trying to provision, there's most likely someone in the community that already has, and can help your out.
