# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'ipaddr'

unless Vagrant.has_plugin?("vagrant-hostmanager")
  raise 'vagrant-hostmanager is not installed! run "vagrant plugin install vagrant-hostmanager" to fix'
end

VAGRANTFILE_API_VERSION = "2"

class IpAssigner

  def self.next_ip(current_ip)
    IPAddr.new(current_ip).succ.to_s
  end

  def self.generate(start_ip, number_of_addresses)
    number_of_addresses.times.inject([start_ip]) { |acc, i|
      nekst = next_ip(acc.last)
      acc + [nekst]
    }
  end

end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.box = 'hfm4/centos7'
  config.ssh.insert_key = false
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true

  accept_oracle_licence = true # set to false if you don't agree (will not install Java8 for you)

  private_network_begin =  "192.168.5.100" # private ip will start incrementing from this
  zk_vm_memory_mb = 256
  zk_port = 2181
  kafka_vm_memory_mb = 512
  kafka_port = 9092

  # < ------- These need to be set in group vars if using Ansible w/o Vagrant -------

  # The follwing variables will need to be passed manually if you want to use the Ansible
  # playbooks w/o Vagrant. They are set in this Vagrantfile in this manner because it allows us to easily
  # increase or decrease the cluster sizes.

  # Note that zk_id must be unique for each host in the cluster. It should ideally not change
  # throughout the lifetime of the Zookeeper installation on a given machine.
  zk_cluster_info = {
    'zk-node-1' => { :zk_id => 1 },
    'zk-node-2' => { :zk_id => 2 },
    'zk-node-3' => { :zk_id => 3 }
  }

  # Note that broker_id must be unique for each host in the cluster. It should ideally not change
  # throughout the lifetime of the Kafka installation on a given machine.
  kafka_cluster_info = {
    'kafka-node-1' => { :broker_id => 1 },
    'kafka-node-2' => { :broker_id => 2 },
    'kafka-node-3' => { :broker_id => 3 }
  }

  ## ------- These need to be set in group vars if using Ansible w/o Vagrant ------- >

  # Helper to make new ips
  zk_ips = IpAssigner.generate(
    private_network_begin,
    zk_cluster_info.size)

  kafka_ips = IpAssigner.generate(
    IpAssigner.next_ip(zk_ips.last || private_network_begin),
    kafka_cluster_info.size)

  zk_cluster = Hash[zk_cluster_info.map.with_index { |(k, v), idx|
    [k, v.merge(
      :ip => zk_ips[idx],
      :memory => zk_vm_memory_mb,
      :client_port => zk_port,
      :client_forward_to => zk_port + idx )]
  }]

  kafka_cluster = Hash[kafka_cluster_info.map.with_index { |(k, v), idx|
    [k, v.merge(
      :ip => kafka_ips[idx],
      :memory => kafka_vm_memory_mb,
      :client_port => kafka_port,
      :client_forward_to => kafka_port + idx )]
  }]

  total_cluster = zk_cluster.merge(kafka_cluster)

  total_cluster.each_with_index do |(short_name, info), idx|

    config.vm.define short_name do |host|
      host.vm.network :forwarded_port, guest: info[:client_port], host: info[:client_forward_to]
      host.vm.network :private_network, ip: info[:ip]
      host.vm.hostname = short_name
      host.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", info[:memory]]
      end

      # This allows us to provision everything in one go, in parallel.
       if idx == (total_cluster.size - 1)
         host.vm.provision :ansible do |ansible|
           ansible.playbook = "site.yml"
           ansible.groups = {
             "zk" => zk_cluster.keys,
             "kafka" => kafka_cluster.keys
           }
           ansible.verbose = 'vv'
           ansible.sudo = true
           ansible.limit = 'all' # otherwise, Ansible only runs on the current host...
           ansible.extra_vars = {
             accept_oracle_licence: accept_oracle_licence,
             vagrant_zk_client_port: zk_port,
             vagrant_zk_cluster_info: zk_cluster_info,
             vagrant_kafka_cluster_info: kafka_cluster_info
           }
         end
       end

    end

  end

end
