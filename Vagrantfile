# -*- mode: ruby -*-
# vi: set ft=ruby :
#vagrant plugins required:
#vagrant-auto_network, vagrant-hostsupdater
require 'yaml'
VAGRANTFILE_API_VERSION = "2"

# Way to Check if needed plugins are installed.
def check_plugin_installed(name)
  if !Vagrant.has_plugin?(name)
    puts "plugin missing #{name}"
    puts "please run \"vagrant plugin install #{name}\""
    exit(1)
  end
end

# Cross-platform way of finding an executable in the $PATH.
def which(cmd)
  exts = ENV['PATHEXT'] ? ENV['PATHEXT'].split(';') : ['']
  ENV['PATH'].split(File::PATH_SEPARATOR).each do |path|
    exts.each { |ext|
      exe = File.join(path, "#{cmd}#{ext}")
      return exe if File.executable?(exe) && !File.directory?(exe)
    }
  end
  return nil
end

# plugins check
check_plugin_installed("vagrant-auto_network")
check_plugin_installed("vagrant-hostsupdater")

# Use config.yml for basic VM configuration.
dir = File.dirname(File.expand_path(__FILE__))
if !File.exist?("#{dir}/config.yml")
  raise 'Configuration file not found! Please copy example.config.yml to config.yml and try again.'
end
vconfig = YAML::load_file("#{dir}/config.yml")

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.hostname = vconfig['vagrant_hostname']
  if vconfig['vagrant_ip'] == "0.0.0.0"
    config.vm.network :private_network, :ip => vconfig['vagrant_ip'], :auto_network => true
  else
    config.vm.network :private_network, ip: vconfig['vagrant_ip']
  end

  if !vconfig['vagrant_public_ip'].empty? && vconfig['vagrant_public_ip'] == "0.0.0.0"
    config.vm.network :public_network
  elsif !vconfig['vagrant_public_ip'].empty?
    config.vm.network :public_network, ip: vconfig['vagrant_public_ip']
  end

  config.ssh.insert_key = false
  config.ssh.forward_agent = true

  config.vm.box = vconfig['vagrant_box']

  # Add all server names as aliases.
  config.hostsupdater.aliases = []
  # Add all hosts that aren't defined as Ansible vars.
  if vconfig['openapphackvm_webserver'] == "apache"
    for host in vconfig['apache_vhosts']
      unless host['servername'].include? "{{"
        config.hostsupdater.aliases.push(host['servername'])
      end
    end
  else
    for host in vconfig['nginx_hosts']
      unless host['server_name'].include? "{{"
        config.hostsupdater.aliases.push(host['server_name'])
      end
    end
  end

  for synced_folder in vconfig['vagrant_synced_folders'];
    config.vm.synced_folder synced_folder['local_path'], synced_folder['destination'],
      type: synced_folder['type'],
      rsync__auto: "true",
      rsync__exclude: synced_folder['excluded_paths'],
      rsync__args: ["--verbose", "--archive", "--delete", "-z", "--chmod=ugo=rwX"],
      id: synced_folder['id'],
      create: synced_folder.include?('create') ? synced_folder['create'] : false,
      mount_options: synced_folder.include?('mount_options') ? synced_folder['mount_options'] : []
  end

  # Provision using Ansible provisioner if Ansible is installed on host.

  if which('ansible-playbook')
    config.vm.provision "ansible" do |ansible|
      ansible.playbook = "#{dir}/provisioning/playbook.yml"
      ansible.sudo = true
    end
  else
    config.vm.provision "ansible_local" do |ansible|
      ansible.playbook = "provisioning/playbook.yml"
      ansible.galaxy_role_file = "provisioning/requirements.yml"
    end
  end


  # VirtualBox.
  config.vm.provider :virtualbox do |v|
    v.name = vconfig['vagrant_hostname']
    v.memory = vconfig['vagrant_memory']
    v.cpus = vconfig['vagrant_cpus']
    v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    v.customize ["modifyvm", :id, "--ioapic", "on"]
  end

 # TODO Optional Include Docker provisioner


  config.vm.define vconfig['vagrant_machine_name'] do |d|
  end
end
