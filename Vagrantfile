# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'
require 'vagrant/ui'


UI = Vagrant::UI::Colored.new
box = 'generic/centos7'
settings = YAML.load_file 'vagrantConf.yml'
http_proxy = ENV['HTTP_PROXY'] || ENV['http_proxy'] ||  ""
https_proxy = ENV['HTTPS_PROXY'] || ENV['https_proxy'] ||  ""
no_proxy = 'localhost,127.0.0.1,' + settings.map { |_, v| "#{v['ip']},#{v['name']}" }.join(',')

# create dynamic inventory file. ansible provisioner 's dynamic inventory got some bugs
inventory_file = 'hosts'
File.open(inventory_file, 'w') do |f|
  settings.each do |_, machine_info|
    f.puts(machine_info['name'] + ' ' + 'ansible_host=' + machine_info['ip'] + ' ' + 'ip=' + machine_info['ip']) 
  end

  %w(kube-master etcd kube-node util-node).each do |section|
    f.puts("[#{section}]")
    settings.each do |_, machine_info|
      types = machine_info['type'].split(',') 
      types.each do |t|
        f.puts(machine_info['name']) if t == section 
      end
    end
    f.puts('')
  end
  f.write("[k8s-cluster:children]\nkube-master\nkube-node")

  if https_proxy
    f.puts('')
    f.puts("[all:vars]")
    f.puts("http_proxy=#{http_proxy}")
    f.puts("https_proxy=#{https_proxy}")
    f.puts("install_dnsmasq=true")
  end
end


Vagrant.configure('2') do |config|
  config.vm.box = "#{box}"
  config.ssh.insert_key = false
  config.ssh.forward_agent = true
  config.ssh.forward_x11 = true
  config.vm.synced_folder './', '/vagrant'

  required_plugins = %w( vagrant-hostmanager vagrant-cachier vagrant-vbguest vagrant-proxyconf)
  required_plugins.each do |plugin|
    exec "vagrant plugin install #{plugin};vagrant #{ARGV.join(' ')}" unless Vagrant.has_plugin?(plugin) || ARGV[0] == 'plugin'
  end
  
  config.hostmanager.enabled = true
  config.hostmanager.manage_guest = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true
  config.cache.scope = :box # :machine
  config.vbguest.auto_update = false

  if Vagrant.has_plugin?('vagrant-proxyconf') && https_proxy
   config.proxy.http = http_proxy
   config.proxy.https = https_proxy
   config.proxy.no_proxy = no_proxy
   UI.info "https_proxy: #{config.proxy.https}"
  end

  settings.each do |name, machine_info|
    config.vm.define name do |node|
      node.vm.hostname = machine_info['name']
      node.vm.network :private_network, ip: machine_info['ip']
      !machine_info['box'].nil? && node.vm.box = machine_info['box']

      node.vm.provider 'virtualbox' do |vb|
        vb.linked_clone = true
        vb.cpus = machine_info['cpus']
        vb.memory = machine_info['mem']
        vb.customize ['modifyvm', :id, '--natdnshostresolver1', 'on']
        # for dcos ntptime
        vb.customize ['guestproperty', 'set', :id, '/VirtualBox/GuestAdd/VBoxService/--timesync-set-threshold', 1000]
      end

      if !machine_info['ansible_host'].nil? && machine_info['ansible_host']
        #node.vm.network 'forwarded_port', guest: 6443, host: 443
        ssh_prv_key = File.read("#{Dir.home}/.vagrant.d/insecure_private_key")
        UI.info 'Insert vagrant insecure key to bootstreap node...', bold: true
        node.vm.provision 'shell' do |sh|
          sh.inline = <<-SHELL
            [ ! -e /home/vagrant/.ssh/id_rsa ] && echo "#{ssh_prv_key}" > /home/vagrant/.ssh/id_rsa && chown vagrant:vagrant /home/vagrant/.ssh/id_rsa && chmod 600 /home/vagrant/.ssh/id_rsa
            echo Provisioning of ssh keys completed [Success].
          SHELL
        end

        node.vm.provision 'shell', inline: 'yum install -y python-pip'
        node.vm.provision :ansible_local do |ansible|
          #ansible.install_mode = :pip # or default( by os package manager)
          ansible.install_mode = "pip"
          ansible.version = "2.9.11"
          ansible.config_file = 'ansible.cfg'
          ansible.inventory_path = 'hosts'
          ansible.become = true
          ansible.limit = 'all'
          ansible.playbook = 'playbook.yml'
          ansible.verbose = 'true'
        end
      end
    end
  end
end
