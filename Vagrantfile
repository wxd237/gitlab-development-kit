# -*- mode: ruby -*-
# vi: set ft=ruby :

# Please see the Vagrant section in the readme for caveats and tips
# https://gitlab.com/gitlab-org/gitlab-development-kit/tree/master#vagrant

VAGRANTFILE_API_VERSION = "2"

def running_in_admin_mode?
    return false unless Vagrant::Util::Platform.windows?
    
    (`reg query HKU\\S-1-5-19 2>&1` =~ /ERROR/).nil? 
end

if Vagrant::Util::Platform.windows? && !running_in_admin_mode?
	raise Vagrant::Errors::VagrantError.new, "You must run the GitLab Vagrant from an elevated command prompt"
end

required_plugins = %w(vagrant-share)
required_plugins_non_windows = %w(facter)
required_plugins_windows = %w() # %w(vagrant-winnfsd) if https://github.com/GM-Alex/vagrant-winnfsd/issues/50 gets fixed

if Vagrant::Util::Platform.windows?
	required_plugins.concat required_plugins_windows
else
	required_plugins.concat required_plugins_non_windows
end

# thanks to http://stackoverflow.com/a/28801317/1233435
required_plugins.each do |plugin|
	need_restart = false
	unless Vagrant.has_plugin? plugin
		system "vagrant plugin install #{plugin}"
		need_restart = true
	end
	exec "vagrant #{ARGV.join(' ')}" if need_restart
end

$apt_reqs = <<EOT
apt-get update
apt-get -y install git postgresql libpq-dev phantomjs redis-server libicu-dev cmake g++ nodejs libkrb5-dev
EOT

# CentOS 6 kernel doesn't suppose UID mapping (affects vagrant-lxc mostly).
$user_setup = <<EOT
if [ $(id -u vagrant) != $(stat -c %u /vagrant) ]; then
	useradd -u $(stat -c %u /vagrant) -m build
	echo "build ALL=(ALL) NOPASSWD:ALL" | tee /etc/sudoers.d/build
	DEV_USER=build
else
	DEV_USER=vagrant
fi
sudo -u $DEV_USER -i bash -c "gpg --keyserver hkp://keys.gnupg.net --recv-keys D39DC0E3"
sudo -u $DEV_USER -i bash -c "curl -sSL https://get.rvm.io | bash -s stable --ruby=2.1.6"
sudo -u $DEV_USER -i bash -c "gem install bundler"
sudo chown -R $DEV_USER:$DEV_USER /home/vagrant
sudo -u $DEV_USER -i bash -c "cp -r /vagrant/* /home/vagrant/gitlab-development-kit/"

# automatically move into the gitlab-development-kit folder, but only add the command
# if it's not already there
sudo -u $DEV_USER -i bash -c "grep -q 'cd /home/vagrant/gitlab-development-kit/' /home/vagrant/.bash_profile || echo 'cd /home/vagrant/gitlab-development-kit/' >> /home/vagrant/.bash_profile"
EOT

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
	config.vm.box = "ubuntu/trusty64"
	config.vm.provision "shell", inline: $apt_reqs
	config.vm.provision "shell", inline: $user_setup

	if !Vagrant::Util::Platform.windows?
		# NFS setup
		config.vm.network "private_network", type: "dhcp"
	end

	# paths must be listed as shortest to longest per bug: https://github.com/GM-Alex/vagrant-winnfsd/issues/12#issuecomment-78195957
	config.vm.synced_folder ".", "/vagrant", :nfs => !Vagrant::Util::Platform.windows?
	config.vm.synced_folder "gitlab/", "/home/vagrant/gitlab-development-kit/gitlab", :create => true, :nfs => !Vagrant::Util::Platform.windows?
	config.vm.synced_folder "gitlab-ci/", "/home/vagrant/gitlab-development-kit/gitlab-ci", :create => true, :nfs => !Vagrant::Util::Platform.windows?
	config.vm.synced_folder "gitlab-shell/", "/home/vagrant/gitlab-development-kit/gitlab-shell", :create => true, :nfs => !Vagrant::Util::Platform.windows?
	config.vm.synced_folder "gitlab-runner/", "/home/vagrant/gitlab-development-kit/gitlab-runner", :create => true, :nfs => !Vagrant::Util::Platform.windows?

	config.vm.network "forwarded_port", guest: 3000, host: 3000

	config.vm.provider "lxc" do |v, override|
		override.vm.box = "fgrehm/trusty64-lxc"
	end
	config.vm.provider "virtualbox" do |vb|
		if Vagrant::Util::Platform.windows?
			# thanks to https://github.com/rdsubhas/vagrant-faster/blob/master/lib/vagrant/faster/action.rb
			# current bug in Facter requires detecting Windows core count seperately - https://tickets.puppetlabs.com/browse/FACT-959
			cpus = `wmic cpu Get NumberOfCores`.split[1].to_i
			# current bug in Facter requires detecting Windows memory seperately - https://tickets.puppetlabs.com/browse/FACT-960
			mem = `wmic computersystem Get TotalPhysicalMemory`.split[1].to_i / 1024 / 1024
		else
			cpus = Facter.value('processors')['count']
			mem = Facter.value('memory').slice! " GiB".to_i * 1024
		end
		
		# use 1/4 of memory or 2 GB, whichever is greatest
		mem = [mem / 4, 2048].max

		# performance tweaks
		# per https://www.virtualbox.org/manual/ch03.html#settings-processor set cpus to real cores, not hyperthreads
		vb.cpus = cpus
		vb.memory = mem
		vb.customize ["modifyvm", :id, "--nestedpaging", "on"]
		vb.customize ["modifyvm", :id, "--largepages", "on"]
		if cpus > 1
			vb.customize ["modifyvm", :id, "--ioapic", "on"]
		end

		# uncomment if you don't want to use all of host machines CPU
		#vb.customize ["modifyvm", :id, "--cpuexecutioncap", "50"]

		# uncomment if you need to troubleshoot using a GUI
		#vb.gui = true
	end
end