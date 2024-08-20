Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-22.04"

  (1..4).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.hostname = "node#{i}"
      node.vm.network "private_network", ip: "192.168.50.#{200 + i - 1}"
      node.vm.provider "vmware_desktop" do |v|
        v.vmx["memsize"] = "3096"
        v.vmx["numvcpus"] = "2"
      end
      node.vm.provision "shell", inline: <<-SHELL
        apt-get update
        apt-get install -y containerd openssh-server sshpass
        systemctl start containerd
        systemctl enable containerd

        # Disable swap
        swapoff -a
        sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

        # Enable kernel modules
        modprobe overlay
        modprobe br_netfilter
        echo 'net.bridge.bridge-nf-call-iptables = 1' | sudo tee -a /etc/sysctl.d/k8s.conf
        echo 'net.bridge.bridge-nf-call-ip6tables = 1' | sudo tee -a /etc/sysctl.d/k8s.conf
        echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/k8s.conf
        sysctl --system

        # Setup SSH key
        ssh-keygen -t rsa -N '' -f /home/vagrant/.ssh/id_rsa
        cat /home/vagrant/.ssh/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys
        chown vagrant:vagrant /home/vagrant/.ssh/authorized_keys

        # Update /etc/hosts
        echo "192.168.50.200 node1" >> /etc/hosts
        echo "192.168.50.201 node2" >> /etc/hosts
        echo "192.168.50.202 node3" >> /etc/hosts
        echo "192.168.50.203 node4" >> /etc/hosts

        # Install containerd
        mkdir -p /etc/containerd
        containerd config default | tee /etc/containerd/config.toml
        sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml
        systemctl restart containerd
        systemctl enable containerd

        # Install ansible
        apt-get install software-properties-common -y
        add-apt-repository --yes --update ppa:ansible/ansible
        apt-get update
        apt-get install ansible -y

        echo "[control_plane]" >> /etc/ansible/hosts
        echo "192.168.50.200" >> /etc/ansible/hosts
        echo "" >> /etc/ansible/hosts
        echo "[worker_nodes]" >> /etc/ansible/hosts
        echo "192.168.50.201" >> /etc/ansible/hosts
        echo "192.168.50.202" >> /etc/ansible/hosts
        echo "192.168.50.203" >> /etc/ansible/hosts
        echo "" >> /etc/ansible/hosts
        echo "[kubernetes:children]" >> /etc/ansible/hosts
        echo "control_plane" >> /etc/ansible/hosts
        echo "worker_nodes" >> /etc/ansible/hosts


        # Install dependencies for Kubernetes
        curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
        echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list

        # Install Kubernetes packages - kubeadm, kubelet and kubectl
        #Add k8s.io's apt repository gpg key, this will likely change for each version of kubernetes release. 
        apt-get install -y apt-transport-https ca-certificates curl gpg
        curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

        # Add the Kubernetes apt repository
        echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

        # Update the package list and use apt-cache policy to inspect versions available in the repository
        apt-get update
        apt-cache policy kubelet | head -n 20

        #Install the required packages, if needed we can request a specific version. 
        VERSION=1.29.1-1.1
        apt-get install -y kubelet=$VERSION kubeadm=$VERSION kubectl=$VERSION
        apt-mark hold kubelet kubeadm kubectl containerd

        #1 - systemd Units
        #Check the status of our kubelet and our container runtime, containerd.
        #The kubelet will enter a inactive (dead) state until a cluster is created or the node is joined to an existing cluster.
        systemctl status kubelet.service
        systemctl status containerd.service
      SHELL
    end
  end
end
