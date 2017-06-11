# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.box = "ubuntu/trusty64"
  config.vm.box_url = "https://atlas.hashicorp.com/ubuntu/boxes/trusty64"
  config.vm.define "dse-node"
  config.vm.hostname = "dse-node"
  config.vm.network "private_network", ip: "192.168.56.10"
  config.vm.boot_timeout = 30000
  config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'"

  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
    vb.memory = 32768
    vb.cpus = 4
  end

  config.ssh.insert_key = false
  config.vm.provision "file", source: "~/.ssh/id_rsa", destination: "~/.ssh/authorized_keys"
  config.vm.provision :shell,
    :keep_color => true,
    :path => "setup.sh"
end
