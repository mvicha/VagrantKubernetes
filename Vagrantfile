# -*- mode: ruby -*-
# vi: set ft=ruby :

# Join command
JOIN_COMMAND = ""
servers = [
  {
    :name => "kubemaster",
    :type => "master",
    :box => "debian/stretch64",
    #:eth1 => "192.168.86.10",
    :eth1 => "192.168.122.254",
    :mem => "2048",
    :cpu => "2"
  },
  {
    :name => "kubenode1",
    :type => "node",
    :box => "debian/stretch64",
    #:eth1 => "192.168.86.11",
    :eth1 => "192.168.122.2",
    :mem => "1024",
    :cpu => "1"
  },
  {
    :name => "kubenode2",
    :type => "node",
    :box => "debian/stretch64",
    #:eth1 => "192.168.86.12",
    :eth1 => "192.168.122.3",
    :mem => "1024",
    :cpu => "1"
  },
  {
    :name => "kubenode3",
    :type => "node",
    :box => "debian/stretch64",
    #:eth1 => "192.168.86.13",
    :eth1 => "192.168.122.4",
    :mem => "1024",
    :cpu => "1"
  }
]

$lastnode = "kubenode3"

$post_up_message = <<MSG
Login to master and execute:
bash post_install.sh
MSG

# This script to install k8s using kubeadm will get executed after a box is provisioned
$configureBox = <<-SCRIPT

  # install all the required packages
  apt-get update
  apt-get install -y apt-transport-https ca-certificates curl software-properties-common net-tools bash-completion git vim

  sudo update-alternatives --set editor /usr/bin/vim.basic

  # install docker latest stable
  curl -fsSL https://get.docker.com | sh

  # run docker commands as vagrant user (sudo not required)
  usermod -aG docker vagrant

  # Update kubelet config to use cgroupfs
  echo "KUBELET_EXTRA_ARGS=--cgroup-driver=cgroupfs" > /etc/default/kubelet

  # install kubeadm
  apt-get install -y apt-transport-https curl
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
  cat <<'EOF' >/etc/apt/sources.list.d/kubernetes.list
  deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
  apt-get update
  apt-get install -y kubelet kubeadm kubectl
  apt-mark hold kubelet kubeadm kubectl

  # kubelet requires swap off
  swapoff -a

  # ip of this box
  IP_ADDR=`sudo ifconfig eth1 | grep netmask | awk '{print $2}'| cut -f2 -d:`
  # set node-ip
  sudo sed -i "/^[^#]*KUBELET_EXTRA_ARGS=/c\KUBELET_EXTRA_ARGS=--node-ip=$IP_ADDR" /etc/default/kubelet
  sudo systemctl restart kubelet
  kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl
SCRIPT

$configureMaster = <<-SCRIPT
  echo "This is master"
  # ip of this box
  IP_ADDR=`sudo ifconfig eth1 | grep netmask | awk '{print $2}'| cut -f2 -d:`

  # install k8s master
  HOST_NAME=$(hostname -s)
  kubeadm init --apiserver-advertise-address=$IP_ADDR --apiserver-cert-extra-sans=$IP_ADDR  --node-name $HOST_NAME --pod-network-cidr=10.17.0.0/16

  #copying credentials to regular user - vagrant
  sudo --user=vagrant mkdir -p /home/vagrant/.kube
  cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
  chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

  # install Calico pod network addon
  export KUBECONFIG=/etc/kubernetes/admin.conf
  kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml
  wget https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
  sed -i -e 's/192\.168\.0\.0/10\.17\.0\.0/g' calico.yaml
  kubectl apply -f calico.yaml
  #kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml

  # get the join command for nodes
  # save the command to a file for future execution
  kubeadm token create --print-join-command >> /etc/kubeadm_join_cmd.sh
  chmod +x /etc/kubeadm_join_cmd.sh

  # required for setting up password less ssh between guest VMs
  sudo sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
  sudo service sshd restart

  mkdir /root/.kube
  cp /etc/kubernetes/admin.conf /root/.kube/config

  # Install helm
  curl https://raw.githubusercontent.com/helm/helm/master/scripts/get | bash

  # Create helm serviceaccount
  kubectl create serviceaccount tiller --namespace=kube-system

  # Helm Roles
  kubectl create clusterrolebinding tiller-admin --serviceaccount=kube-system:tiller --clusterrole=cluster-admin

  # Init helm
  sudo -u vagrant helm init --service-account=tiller
  sudo -u vagrant helm repo update

  # Install cert-manager
  kubectl create namespace cert-manager
  kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true
  kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.7/deploy/manifests/cert-manager.yaml

  # Install metrics-server
  git clone https://github.com/kubernetes-incubator/metrics-server.git
  cat <<'EOF' >> metrics-server/deploy/1.8+/metrics-server-deployment.yaml
        command:
          - /metrics-server
          - --v=2
          - --kubelet-insecure-tls
          - --kubelet-preferred-address-types=InternalIP
EOF
  kubectl apply -f metrics-server/deploy/1.8+/

  # Install kubernetes-dashboard
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended/kubernetes-dashboard.yaml
  # Grant permissions to access other namespaces
  kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard

  # Create ingress rule
  openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout nginx-selfsigned.key -out nginx-selfsigned.crt -subj "/CN=kubemaster/O=Ciklum Digital"

  cat <<'EOF' >kubernetes-dashboard-ingress-tls.yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/tls
metadata:
  name: kubernetes-dashboard-ingress-tls
  namespace: kube-system
data:
  tls.crt: nginx_selfsigned.crt
  tls.key: nginx_selfsigned.key
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.org/ssl-backends: "kubernetes-dashboard"
    kubernetes.io/ingress.allow-http: "false"
    nginx.ingress.kubernetes.io/secure-backends: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    nginx.ingress.kubernetes.io/configuration-snippet: |
      rewrite ^(/dashboard)$ $1/ permanent;
  name: kubernetes-dashboard-ingress
  namespace: kube-system
spec:
  tls:
  - hosts:
    - kubemaster
    secretName: kubernetes-dashboard-ingress-tls
  rules:
  - host: kubemaster
    http:
      paths:
      - path: /dashboard/?(.*)
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
EOF
  nginx_selfsigned_crt=$(cat nginx-selfsigned.crt | base64 | tr -d "\n")
  nginx_selfsigned_key=$(cat nginx-selfsigned.key | base64 | tr -d "\n")
  sed -i 's/nginx_selfsigned.crt/'''$nginx_selfsigned_crt'''/g' kubernetes-dashboard-ingress-tls.yaml
  sed -i 's/nginx_selfsigned.key/'''$nginx_selfsigned_key'''/g' kubernetes-dashboard-ingress-tls.yaml

  cat <<'EOF' > post_install.sh
# Configure helm client
helm init --client-only --service-account=tiller
helm repo update

# Install nginx-ingress using helm
helm install stable/nginx-ingress --name my-nginx --set rbac.create=true
kubectl patch service my-nginx-nginx-ingress-controller -p '{"spec":{"externalIPs":["192.168.122.254"]}}'
kubectl apply -f kubernetes-dashboard-ingress-tls.yaml
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep kubernetes-dashboard | awk '{print $1}') | grep "^token:"
EOF

SCRIPT

$configureNode = <<-SCRIPT
  echo "This is worker"
  apt-get install -y sshpass
  #sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@kubemaster:/etc/kubeadm_join_cmd.sh .
  sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@192.168.122.254:/etc/kubeadm_join_cmd.sh .

  mkdir /home/vagrant/.kube
  #sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@kubemaster:.kube/config /home/vagrant/.kube/config
  sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@192.168.122.254:.kube/config /home/vagrant/.kube/config
  chown -R vagrant:vagrant /home/vagrant/.kube

  sh ./kubeadm_join_cmd.sh
SCRIPT

$disableSwap = <<-SCRIPT
  # keep swap off after reboot
  sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
SCRIPT

$configureOutputMessage = <<-SCRIPT
  echo "Login to master and execute:"
  echo "bash post_install.sh"
SCRIPT

Vagrant.configure("2") do |config|
  config.vm.post_up_message = $post_up_message
  servers.each do |opts|
    config.vm.define opts[:name] do |config|

      config.vm.box = opts[:box]
      config.vm.hostname = opts[:name]
      #if opts[:type] == "master"
      #  config.vm.network "forwarded_port", guest: 80, host: 80, auto_correct: true
      #  config.vm.network "forwarded_port", guest: 443, host: 443, auto_correct: true
      #end
      #config.vm.network :private_network, ip: opts[:eth1]

      if opts[:type] == "master"
        config.vm.network "forwarded_port", guest: 443, host: 443
        config.vm.network "forwarded_port", guest: 80, host: 81
      end
      config.vm.network :private_network, ip: opts[:eth1]
      #config.vm.network :public_network, bridge: "en0: Wi-Fi (AirPort)", auto_configure: true

      config.vm.provider "virtualbox" do |v|
        v.name = opts[:name]
          #v.customize ["modifyvm", :id, "--groups", "/Ballerina Development"]
          v.customize ["modifyvm", :id, "--memory", opts[:mem]]
           v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
      end

      config.vm.provision "shell", inline: $configureBox

      if opts[:type] == "master"
        config.vm.provision "shell", inline: $configureMaster
      else
        config.vm.provision "shell", inline: $configureNode
      end

      config.vm.provision "shell", inline: $disableSwap

      if opts[:name] == $lastnode
        config.vm.provision "shell", inline: $configureOutputMessage
      end
    end
  end

end

