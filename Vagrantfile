# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "hashicorp/bionic64"

  config.vm.network "private_network", type: "dhcp"
  config.vm.hostname = "mysite"

  config.vm.synced_folder ".", "/vagrant"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "4096"
    vb.cpus = "3"
  end

  config.vm.provision "docker", type: "shell", inline: <<-SHELL
    apt-get update
      
    # https://github.com/docker/docker-install
    curl -sSL https://get.docker.com | sh
    usermod -aG docker vagrant
  SHELL

  config.vm.provision "jekyll", type: "shell", privileged: false, inline: <<-SHELL
    # https://rvm.io/rvm/install
    sudo apt-get update
    sudo apt-get install -y gnupg2
    gpg2 \
      --keyserver hkp://pool.sks-keyservers.net \
      --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
    curl -sSL https://get.rvm.io | bash -s stable --ruby

    source $HOME/.rvm/scripts/rvm

    cd /vagrant
    gem install jekyll bundler
    bundle install
  SHELL
end
