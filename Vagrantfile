IMAGE_NAME = "ubuntu"
COUNTER = 2
Vagrant.configure("2") do |config|
  config.vm.box = IMAGE_NAME
  config.vm.provider "virtualbox" do |v|
    v.memory = 1024
    v.cpus = 2
  end

  config.vm.provision "shell", privileged: true, inline: <<-SCRIPT
      sudo swapoff -a
      sudo sed -i '/swap/d' /etc/fstab
      sudo apt-get update
      sudo apt-get install -y docker.io apt-transport-https curl
      sudo systemctl start docker
      sudo systemctl enable docker
      sudo apt-get update
      sudo apt-get install -y apt-transport-https
      curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
      sudo touch /etc/apt/sources.list.d/kubernetes.list
      echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
      sudo apt-get update
      sudo apt-get install -y kubeadm 

  SCRIPT

  config.vm.define "k8s-master" do |master|
    master.vm.box = IMAGE_NAME
    master.vm.network "private_network", ip: "10.0.0.10"
    master.vm.hostname = "k8s-master"
    master.vm.provision "shell", inline: <<-SHELL
      OUTPUT_FILE=/vagrant/join.sh
      rm -rf /vagrant/join.sh
      sudo kubeadm init --apiserver-advertise-address=10.0.0.10 --pod-network-cidr=10.244.0.0/16  
      sudo kubeadm token create --print-join-command > /vagrant/join.sh
      chmod +x $OUTPUT_FILE
      mkdir -p $HOME/.kube  
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config  
      kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml  
      SHELL
  end

  (1..COUNTER).each do |i|
    config.vm.define "node-#{i}" do |node|
        node.vm.box = IMAGE_NAME
        node.vm.network "private_network", ip: "10.0.0.#{i + 11}"
        node.vm.hostname = "node-#{i}"
        node.vm.provision :shell, privileged: true, inline: <<-SHELL
        sudo /vagrant/join.sh
        echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=10.0.0.#{i + 11}"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
        sudo systemctl daemon-reload
        sudo systemctl restart kubelet
        SHELL
    end
  end
end
