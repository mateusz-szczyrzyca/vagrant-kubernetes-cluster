IMAGE_NAME = "bento/ubuntu-18.04"
N = 3

Vagrant.configure("2") do |config|
    config.ssh.insert_key = false

    config.vm.provider "virtualbox" do |v|
        v.memory = 5120
    end
      
    config.vm.define "k8s-master" do |master|
        master.vm.box = IMAGE_NAME
        master.vm.provider "virtualbox" do |vb|
            vb.customize ["modifyvm", :id, "--cpus", "2"]
        end
        master.vm.network "private_network", ip: "15.0.0.5"
        master.vm.hostname = "k8s-master"
        master.vm.provision "ansible" do |ansible|
            ansible.playbook = "ansible-kubernetes-setup/master-playbook.yml"
            ansible.extra_vars = {
                node_ip: "15.0.0.11",
            }
        end
    end

    (1..N).each do |i|
        config.vm.define "node-#{i}" do |node|
            node.vm.box = IMAGE_NAME
            node.vm.provider "virtualbox" do |vb|
                vb.customize ["modifyvm", :id, "--cpus", "1"]
            end
            node.vm.network "private_network", ip: "15.0.0.#{i + 10}"
            node.vm.hostname = "node-#{i}"
            node.vm.provision "ansible" do |ansible|
                ansible.playbook = "ansible-kubernetes-setup/node-playbook.yml"
                ansible.extra_vars = {
                    node_ip: "15.0.0.#{i + 10}",
                }
            end
        end
    end
end
