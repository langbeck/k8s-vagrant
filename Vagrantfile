require 'ipaddr'

## Load configurations
require './config'

## Override location
ENV["LC_ALL"] = "en_US.UTF-8"


network_cidr = IPAddr.new K8S_NETWORK_CIDR
master_ip = IPAddr.new K8S_MASTER_IP


# VagrantError = Vagrant::Errors::VagrantError

# if not network_cidr.private?
#   raise VagrantError.new("K8S_NETWORK_CIDR isn't a private network")

# if not network_cidr.include?(master_ip)
#   raise VagrantError.new("K8S_NETWORK_CIDR doesn't includes K8S_MASTER_IP")


Vagrant.configure("2") do |config|
  provisioner = Vagrant::Util::Platform.windows? ? :ansible_local : :ansible

  config.ssh.insert_key = false
  config.vm.box = BOX_BASE

  config.vm.provider :libvirt do |libvirt|
    libvirt.memory  = BOX_RAM_MB
    libvirt.cpus    = BOX_CPU_COUNT
  end

  config.vm.provider :virtualbox do |vbox|
    vbox.memory = BOX_RAM_MB
    vbox.cpus   = BOX_CPU_COUNT
  end

  config.vm.provider :vmware_desktop do |vmware|
    vmware.vmx[:memsize]  = BOX_RAM_MB
    vmware.vmx[:numvcpus] = BOX_CPU_COUNT
  end


  config.vm.define K8S_MASTER_NAME, primary: true do |master|
    node_ip = master_ip.to_string

    master.vm.network "private_network", ip: node_ip
    master.vm.hostname = K8S_MASTER_NAME
    master.vm.provision provisioner do |ansible|
        ansible.playbook = "kubernetes-setup/master-playbook.yaml"
        ansible.extra_vars = {
            network_cidr: "#{network_cidr}/#{network_cidr.prefix}",
            node_name: K8S_MASTER_NAME,
            node_ip: node_ip,
        }
    end

    if K8S_NODE_COUNT <= 0
      # Untaint master node to allow scheduling
      master.vm.provision :shell, privileged: false, inline: "kube-untaint-node"
    end
  end


  (1..K8S_NODE_COUNT).each do |i|
    node_ip = IPAddr.new(master_ip.to_i + i, master_ip.family).to_string
    node_name = "node#{i}"

    config.vm.define node_name do |node|
        node.vm.network "private_network", ip: node_ip
        node.vm.hostname = node_name
        node.vm.provision provisioner do |ansible|
            ansible.playbook = "kubernetes-setup/node-playbook.yaml"
            ansible.extra_vars = {
                master_node_name: K8S_MASTER_NAME,
                node_ip: node_ip,
            }
        end
    end
  end

end
