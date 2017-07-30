# -*- mode: ruby -*-
# vim: expandtab shiftwidth=2 softtabstop=2 ft=ruby

# Load local env config
require 'yaml'
dir = File.dirname(File.expand_path(__FILE__))

# defaults
config = YAML::load_file("#{dir}/config/defaults.yaml")

if File.exist?("#{dir}/config/config.yaml")
  config_settings = YAML::load_file("#{dir}/config/config.yaml")
  config.merge!(config_settings)
end

# The number of masters to provision
$num_masters = (ENV['NUM_MASTERS'] || config["num_masters"]).to_i

# The number of dedicated etcd cluster members to provision
$num_etcd = (ENV['NUM_ETCD'] || config["num_etcd"]).to_i

# The number of nodes to provision
$num_nodes = (ENV['NUM_NODES'] || config["num_nodes"]).to_i

# Determine the OS platform to use
$os_image = ENV['OS_IMAGE'] || config["os_image"]

# Define the number of CPUs
$vm_master_cpus = (ENV['MASTER_CPUS'] || ENV['CPUS'] || config["vm_master_cpus"]).to_i
$vm_etcd_cpus = (ENV['ETCD_CPUS'] || ENV['CPUS'] || config["vm_etcd_cpus"]).to_i
$vm_node_cpus = (ENV['NODE_CPUS'] || ENV['CPUS'] || config["vm_node_cpus"]).to_i

# Define the ammount of RAM
$vm_master_mem = (ENV['MASTER_MEMORY'] || ENV['MEMORY'] || config["vm_master_mem"]).to_i
$vm_etcd_mem = (ENV['ETCD_MEMORY'] || ENV['MEMORY'] || config["vm_etcd_mem"]).to_i
$vm_node_mem = (ENV['NODE_MEMORY'] || ENV['MEMORY'] || config["vm_node_mem"]).to_i

Vagrant.configure("2") do |config|

  config.vm.box = $os_image
  config.vm.synced_folder ".", "/home/vagrant/sync", disabled: true

  ansible_group_masters = []
  ansible_group_etcd = []
  ansible_group_nodes = []

  # Provision masters
  $num_masters.times do |n|
    master_vm_name = "master#{n+1}"
    ansible_group_masters.push(master_vm_name)

    config.vm.define master_vm_name do |vm_instance|
      vm_instance.vm.hostname = master_vm_name
      vm_instance.vm.provider :libvirt do |domain|
        domain.memory = $vm_master_mem
        domain.cpus = $vm_master_cpus
      end
    end
  end

  # Provision dedicated etcd cluster members
  $num_etcd.times do |n|
    etcd_vm_name = "etcd#{n+1}"
    ansible_group_etcd.push(etcd_vm_name)

    config.vm.define etcd_vm_name do |vm_instance|
      vm_instance.vm.hostname = etcd_vm_name
      vm_instance.vm.provider :libvirt do |domain|
        domain.memory = $vm_etcd_mem
        domain.cpus = $vm_etcd_cpus
      end
    end
  end

  # Provision nodes
  $num_nodes.times do |n|
    node_vm_name = "node#{n+1}"
    ansible_group_nodes.push(node_vm_name)

    config.vm.define node_vm_name do |vm_instance|
      vm_instance.vm.hostname = node_vm_name
      vm_instance.vm.provider :libvirt do |domain|
        domain.memory = $vm_node_mem
        domain.cpus = $vm_node_cpus
      end
    end
  end

  # Use shell provisioning to register the hostname in DNS.
  config.vm.provision "shell", inline: <<-SHELL
    nmcli con up "System eth0"
  SHELL
#  config.vm.provision "shell", inline: <<-SHELL
#    lvextend --extents +100%FREE /dev/atomicos/docker-pool
#    nmcli con up "System eth0"
#  SHELL

  # Use ansible provisioning to:
  # - expand the docker-pool to maximum
  # - update the Atomic Host
  config.vm.provision "ansible" do |ansible|
    ansible.limit = "all"
#    ansible.verbose = "v"
   ansible.playbook = "atomic_provision.yaml"
    if $num_etcd > 0
      ansible.groups = {
        "masters" => ["master[1:#{$num_masters}]"],
        "etcd" => ["etcd[1:#{$num_etcd}]"],
        "nodes" => ["node[1:#{$num_nodes}]"],
      }
    else
      ansible.groups = {
        "masters" => ["master[1:#{$num_masters}]"],
        "etcd" => ["master[1:#{$num_masters}]"],
        "nodes" => ["node[1:#{$num_nodes}]"],
      }
    end
  end
end
