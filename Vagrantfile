# -*- mode: ruby -*-
# vi: set ft=ruby :

MACHINES = {
  :'ns01' => {
    :domain => 'internal',
    :box => 'almalinux/9/v9.4.20240805',
    :cpus => 1,
    :memory => 1024,
    :networks => [
      [
        :private_network, {
          :ip => '192.168.50.10',
          :virtualbox__intnet => 'dns'
        }
      ]
    ]
  },
  :'client' => {
    :domain => 'internal',
    :box => 'almalinux/9/v9.4.20240805',
    :cpus => 1,
    :memory => 1024,
    :networks => [
      [
        :private_network, {
          :ip => '192.168.50.15',
          :virtualbox__intnet => 'dns'
        }
      ]
    ]
  }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |host_name, host_config|
    config.vm.define host_name do |host|
      host.vm.box = host_config[:box]
      host.vm.host_name = host_name.to_s + '.' + host_config[:domain].to_s

      host.vm.provider :virtualbox do |vb|
        vb.cpus = host_config[:cpus]
        vb.memory = host_config[:memory]
      end

      host_config[:networks].each do |network|
        host.vm.network(network[0], **network[1])
      end

      if MACHINES.keys.last == host_name
        host.vm.provision :ansible do |ansible|
          ansible.playbook = 'playbook.yml'
          ansible.limit = 'all'
          ansible.compatibility_mode = '2.0'
          ansible.raw_arguments = ['--diff']
        end
      end
    end
  end
end
