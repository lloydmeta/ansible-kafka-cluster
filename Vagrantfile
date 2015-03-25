# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.box = 'hfm4/centos7'
  config.ssh.insert_key = false

  accept_oracle_licence = true # set to false if you don't agree (will not install Java8 for you)

  zk_vm_memory_mb = 256
  zk_port = 2181
  kafka_vm_memory_mb = 1024
  kafka_port = 9092

  # zk_id must be unique for each host in the cluster.
  zk_cluster_unique = {
    'zk-node-1' =>  { :ip => "192.168.5.100", :zk_id => 1 },
    'zk-node-2' => { :ip => "192.168.5.101", :zk_id => 2 },
    'zk-node-3' => { :ip => "192.168.5.102", :zk_id => 3 }
  }

  zk_cluster = zk_cluster_unique.map { |k, v|
    [k, v.merge(:memory => zk_vm_memory_mb, :client_port => zk_port )]
  }

  # broker_id must be unique for each host in the cluster
  kafka_cluster_unique = {
    'kafka-node-1' =>  { :ip => "192.168.5.103", :broker_id => 1 },
    'kafka-node-2' => { :ip => "192.168.5.104", :broker_id => 2 }
  }

  kafka_cluster = kafka_cluster_unique.map { |k, v|
    [k, v.merge(:memory => kafka_vm_memory_mb, :client_port => kafka_port )]
  }

  zk_cluster.each_with_index do |(short_name, info), idx|

    config.vm.define short_name do |host|
      host.vm.network :forwarded_port, guest: info[:client_port], host: info[:client_port] + idx
      host.vm.network :private_network, ip: info[:ip]
      host.vm.hostname = short_name
      host.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", info[:memory]]
      end

    end

  end

  kafka_cluster.each_with_index do |(short_name, info), idx|

    # prevents us from forwarding the port with the same index as ZK cluster...prolly should refactor
    # to be DRYer
    config.vm.define short_name do |host|
      host.vm.network :forwarded_port, guest: info[:client_port], host: info[:client_port] + idx
      host.vm.network :private_network, ip: info[:ip]
      host.vm.hostname = short_name
      host.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", info[:memory]]
      end

      # This allows us to provision everything in one go, in parallel.
       if idx == (kafka_cluster.size - 1)
         host.vm.provision :ansible do |ansible|
           ansible.playbook = "site.yml"
           ansible.groups = {
             "zk" => zk_cluster_unique.keys,
             "kafka" => kafka_cluster_unique.keys
           }
           ansible.inventory_path=".vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory"
           ansible.verbose = 'vv'
           ansible.sudo = true
           ansible.limit = 'all'
           ansible.extra_vars = {
             accept_oracle_licence: accept_oracle_licence,
             zk_cluster_info: zk_cluster,
             kafka_cluster_info: kafka_cluster
           }
         end
       end

    end

  end

end
