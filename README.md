# GitLab Development Kit

The GDK runs a GitLab development environment isolated in a directory.
This environment contains GitLab CE, CI and Runner.
This project uses Foreman to run dedicated Postgres and Redis processes for
GitLab development. All data is stored inside the gitlab-development-kit
directory. All connections to supporting services go through Unix domain
sockets to avoid port conflicts.

## Design goals

- Get the user started, do not try to take care of everything
- Run everything as your 'desktop' user on your development machine
- GitLab Development Kit itself does not run `sudo` commands
- It is OK to leave some things to the user (e.g. installing Ruby)

## Installation

The preferred way to use GitLab Development Kit is to install Ruby and dependencies on your 'native' OS.
We strongly recommend the native install since it is much faster than a virtualized one.
If you want to use [Vagrant](https://www.vagrantup.com/) instead please see [the instructions for our (experimental) Vagrantfile](#vagrant).

### Install dependencies

#### Prerequisites for all platforms

If you do not have the dependencies below you will experience strange errors during installation.

1. A non-root unix user, this can be your normal user but **DO NOT** run the installation as a root user
1. Ruby 2.1.6 installed with a ruby version manager (RVM, ruby-build, rbenv, chruby, etc.), **DO NOT** use the system Ruby
1. bundler, which you can install with `gem install bundler`

#### OS X 10.9

Please read the prerequisites for all platforms.

```
brew tap homebrew/dupes
brew tap homebrew/versions
brew install git redis postgresql phantomjs198 libiconv icu4c pkg-config cmake nodejs
brew link phantomjs198
bundle config build.nokogiri --with-iconv-dir=/usr/local/opt/libiconv
```

#### Ubuntu

Please read the prerequisites for all platforms.

```
sudo apt-get install git postgresql libpq-dev phantomjs redis-server libicu-dev cmake g++ nodejs libkrb5-dev
```

#### Arch Linux

Please read the prerequisites for all platforms.

```
sudo pacman -S postgresql phantomjs redis postgresql-libs icu nodejs ed cmake openssh git
```

#### Debian

Please read the prerequisites for all platforms.

```
sudo apt-get install postgresql libpq-dev redis-server libicu-dev cmake g++ nodejs libkrb5-dev ed
```

You need to install phantomjs manually

```
PHANTOM_JS="phantomjs-1.9.8-linux-x86_64"
cd ~
wget https://bitbucket.org/ariya/phantomjs/downloads/$PHANTOM_JS.tar.bz2
tar -xvjf $PHANTOM_JS.tar.bz2
sudo mv $PHANTOM_JS /usr/local/share
sudo ln -s /usr/local/share/$PHANTOM_JS/bin/phantomjs /usr/local/bin
phantomjs --version
```

#### RedHat

Please read the prerequisites for all platforms.

Please contribute this by sending a merge request.

#### CentOS

Please read the prerequisites for all platforms.

This is tested on CentOS 6.5

```
sudo yum install http://yum.postgresql.org/9.3/redhat/rhel-6-x86_64/pgdg-redhat93-9.3-1.noarch.rpm
sudo yum install http://download.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
sudo yum install postgresql93-server libicu-devel cmake gcc-c++ redis
sudo yum install fontconfig freetype libfreetype.so.6 libfontconfig.so.1 libstdc++.so.6

sudo gpg2 --keyserver hkp://keys.gnupg.net --recv-keys D39DC0E3
sudo curl -sSL https://get.rvm.io | bash -s stable
sudo source /etc/profile.d/rvm.sh
sudo rvm install 2.1
sudo rvm use 2.1
#Ensure your user is in rvm group
sudo usermod -a -G rvm <username>
#add iptables exceptions, or sudo service stop iptables
```
PhantomJS - You will want to download the required version of PhantomJS and place the binary on the path.

Git 1.7.1-3 is the latest git binary for CentOS 6.5 and gitlab.  Spinach tests will fail due to a higher version requirement by gitlab.
You can follow the instructions found [here](https://gitlab.com/gitlab-org/gitlab-recipes/tree/master/install/centos#add-puias-computational-repository)
to install a newer binary version of git.

### Clone GitLab Development Kit repository

```
git clone https://gitlab.com/gitlab-org/gitlab-development-kit.git
cd gitlab-development-kit
```

### Install the repositories and gems

The `Makefile` will clone the repositories, install the Gem bundles and set up
basic configuration files. Pick one:

```
# Clone the official repositories of gitlab and gitlab-shell
make
```

Alternatively, you can clone straight from your forked repositories or GitLab EE.

```
# Clone your own forked repositories
make gitlab_repo=git@gitlab.com:example/gitlab-ce.git gitlab_shell_repo=git@gitlab.com:example/gitlab-shell.git \
  gitlab_ci_repo=git@gitlab.com:example/gitlab-ci.git gitlab_runner_repo=git@gitlab.com:example/gitlab-ci-runner.git
```

### Post-installation

Start Redis and PostgreSQL by running the command below in the root of the project:

    bundle exec foreman start

Keep the above command running and seed the main GitLab database from a new terminal session:

    cd gitlab && bundle exec rake db:create dev:setup

Finally, start the main GitLab rails application in the gitlab subdirectory of the project:

    bundle exec foreman start

Now you can go to http://localhost:3000 in your browser.
The development login credentials are `root` and `5iveL!fe`

If you want to work on GitLab CI, first seed the GitLab CI database:

    cd gitlab-ci && bundle exec rake db:create db:setup

To start the GitLab CI rails application:

    bundle exec foreman start

Setup the GitLab Runner:

    cd gitlab-runner
    CI_SERVER_URL=http://localhost:9000 bundle exec ./bin/setup

Start the GitLab Runner:

    bundle exec ./bin/runner

To enable the OpenLDAP server, see the OpenLDAP instructions in this readme.

END Post-installation

Please do not delete the 'END Post-installation' line above. It is used to
print the post-installation message from the `Makefile`.

### Vagrant

[Vagrant](http://www.vagrantup.com) is a tool for setting up identical development
environments including all dependencies. Vagrant will default to using
[VirtualBox](http://www.virtualbox.org), but it has many plugins for different
environments.

Vagrant allows you to develop GitLab without affecting your host machine (but we 
recommend developing GitLab on metal if you can).
Vagrant can be very slow since the NFS server is on the host OS and GitLab 
(testing) accesses a lot of files.
You can improve the speed by running NFS on the guest OS but in that case you 
should take care to not lose the files when you shut down the VM.

Once you have Vagrant installed, simply type `vagrant up` in this directory. Vagrant
will download an OS image, bring it up, and install all the prerequisites. You then
type `vagrant ssh` to ssh into the box. This directory will be available as a shared
folder in `/vagrant/` and you can continue at 
*[Install the repositories and gems](#install-the-repositories-and-gems)* above.

Typically you keep editing on the host machine but run `make`, `bundle exec`, etc.
inside the `vagrant ssh` session.

Note: On some setups the shared folder will have the wrong user. This is detected
by the Vagrantfile and you should `sudo su - build` to switch to the correct user
in that case.

## Development

When doing development, you will need one shell session (terminal window)
running Postgres and Redis, and one or more other sessions to work on GitLab
itself.

### Example

First start Postgres and Redis.

```
# terminal window 1
# current directory: gitlab-development-kit
bundle exec foreman start
```

Next, start a Rails development server.

```
# terminal window 2
# current directory: gitlab-development-kit/gitlab
bundle exec foreman start
```

Now you can go to http://localhost:3000 in your browser.
The development login credentials are `root` and `5iveL!fe`

### Running the tests

In order to run the test you can use the following commands:
- `rake spinach` to run the spinach suite
- `rake spec` to run the rspec suite
- `rake jasmine:ci` to run the jasmine test suite
- `rake gitlab:test` to run all the tests

Note: You can't run `rspec .` since this will try to run all the `_spec.rb`
files it can find, also the ones in `/tmp`

To run a single test file you can use:

- `bundle exec rspec spec/controllers/commit_controller_spec.rb` for a rspec test
- `bundle exec spinach features/project/issues/milestones.feature` for a spinach test

## Update gitlab and gitlab-shell repositories

When working on a new feature, always check that your `gitlab` repository is up
to date with the upstream master branch.

In order to fetch the latest code, first make sure that `foreman` for
postgres is runnning (needed for db migration) and then run:

```
make update
```

This will update both `gitlab`, `gitlab-ci` and `gitlab-shell` and run any possible migrations.
You can also update them separately by running `make gitlab-update` `make gitlab-ci-update` and
`make gitlab-shell-update` respectively.

## OpenLDAP

To run the OpenLDAP installation included in the GitLab development kit do the following:

```
vim Procfile # remove the comment on the OpenLDAP line
cd gitlab-openldap
make # will setup the databases
```

in the gitlab repository edit config/gitlab.yml;

```yaml
ldap:
  enabled: true
  servers:
    main:
      label: LDAP
      host: 127.0.0.1
      port: 3890
      uid: 'uid'
      method: 'plain' # "tls" or "ssl" or "plain"
      base: 'dc=example,dc=com'
      user_filter: ''
      group_base: 'ou=groups,dc=example,dc=com'
      admin_group: ''
    # Alternative server, multiple LDAP servers only work with GitLab-EE
    # alt:
    #   label: LDAP-alt
    #   host: 127.0.0.1
    #   port: 3890
    #   uid: 'uid'
    #   method: 'plain' # "tls" or "ssl" or "plain"
    #   base: 'dc=example-alt,dc=com'
    #   user_filter: ''
    #   group_base: 'ou=groups,dc=example-alt,dc=com'
    #   admin_group: ''
```

The second database is optional, and will only work with Gitlab-EE.

## Troubleshooting

### Rails cannot connect to Postgres

- Check if foreman is running in the gitlab-development-kit directory.
- Check for custom Postgres connection settings defined via the environment; we
  assume none such variables are set. Look for them with `set | grep '^PG'`.

### 'LoadError: dlopen' when starting Ruby apps

This can happen when you try to load a Ruby gem with native extensions that
were linked against a system library that is no longer there. A typical culprit
is Homebrew on OS X, which encourages frequent updates (`brew update && brew
upgrade`) which may break binary compatibility.

```
bundle exec rake db:create gitlab:setup
rake aborted!
LoadError: dlopen(/Users/janedoe/.rbenv/versions/2.1.2/lib/ruby/gems/2.1.0/extensions/x86_64-darwin-13/2.1.0-static/charlock_holmes-0.6.9.4/charlock_holmes/charlock_holmes.bundle, 9): Library not loaded: /usr/local/opt/icu4c/lib/libicui18n.52.1.dylib
  Referenced from: /Users/janedoe/.rbenv/versions/2.1.2/lib/ruby/gems/2.1.0/extensions/x86_64-darwin-13/2.1.0-static/charlock_holmes-0.6.9.4/charlock_holmes/charlock_holmes.bundle
  Reason: image not found - /Users/janedoe/.rbenv/versions/2.1.2/lib/ruby/gems/2.1.0/extensions/x86_64-darwin-13/2.1.0-static/charlock_holmes-0.6.9.4/charlock_holmes/charlock_holmes.bundle
/Users/janedoe/gitlab-development-kit/gitlab/config/application.rb:6:in `<top (required)>'
/Users/janedoe/gitlab-development-kit/gitlab/Rakefile:5:in `require'
/Users/janedoe/gitlab-development-kit/gitlab/Rakefile:5:in `<top (required)>'
(See full trace by running task with --trace)
```

In the above example, you see that the charlock_holmes gem fails to load
`libicui18n.52.1.dylib`. You can try fixing this by re-installing
charlock_holmes:

```
# in /Users/janedoe/gitlab-development-kit
gem uninstall charlock_holmes
bundle install # should reinstall charlock_holmes
```

### Other problems

Please open an issue on the [GDK issue tracker](https://gitlab.com/gitlab-org/gitlab-development-kit/issues).
