IMAGE_NAME = "bento/ubuntu-18.04"
N = 1
CNI_PLUGIN = "calico"
DOCKER_VERSION = "5:18.09"
K8S_VERSION = "1.15"

$kubeadmConfigCopy = <<-SCRIPT
    echo "copy /etc/kubernetes/admin.conf"
    cp /etc/kubernetes/admin.conf /vagrant
SCRIPT

$k8sAutocomplete = <<-SCRIPT
    sudo apt-get install bash-completion
    echo 'source <(kubectl completion bash)' >>~/.bashrc
SCRIPT

Vagrant.configure("2") do |config|
    config.ssh.insert_key = false

    config.vm.define "k8s-master" do |master|
        master.vm.provider "virtualbox" do |v|
            v.memory = 1500
            v.cpus = 2
        end
        master.vm.box = IMAGE_NAME
        master.vm.network "private_network", ip: "192.168.50.10"
        master.vm.hostname = "k8s-master"
        master.vm.provision "ansible_local" do |ansible|
            ansible.playbook_command = "sudo ANSIBLE_FORCE_COLOR=true ansible-playbook"
            ansible.playbook    = "provision/kubernetes-setup/master-playbook.yml"
            ansible.extra_vars  = {
                node_ip: "192.168.50.10",
                cni_plugin: CNI_PLUGIN,
                docker_version: DOCKER_VERSION,
                k8s_version: K8S_VERSION,
            }
        end
        master.vm.provision "shell", inline: $kubeadmConfigCopy
        master.vm.provision "shell", inline: $k8sAutocomplete
    end

    (1..N).each do |i|
        config.vm.define "node-#{i}" do |node|
            node.vm.provider "virtualbox" do |v|
                v.memory = 1024
                v.cpus = 2
            end
            node.vm.box = IMAGE_NAME
            node.vm.network "private_network", ip: "192.168.50.#{i + 10}"
            node.vm.hostname = "node-#{i}"
            node.vm.provision "ansible_local" do |ansible|
                ansible.playbook_command = "sudo ANSIBLE_FORCE_COLOR=true ansible-playbook"
                ansible.playbook    = "provision/kubernetes-setup/node-playbook.yml"
                ansible.extra_vars  = {
                    node_ip: "192.168.50.#{i + 10}",
                    docker_version: DOCKER_VERSION,
                    k8s_version: K8S_VERSION,
                }
            end
        end
    end
end