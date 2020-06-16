# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.synced_folder ".", "/vagrant"#, type: "virtualbox"
	config.vm.define "master" do |master|
	master.vm.network "private_network", ip: "172.28.128.5"
		master.vm.provider "virtualbox" do |vb|
			vb.memory = "3048"
			vb.name="kube-master"
#			vb.gui = true
			vb.cpus = 2	
		end
   master.vm.hostname = "kube-master"
   master.vm.provision 'shell', inline: <<-EOF
   
    sudo su
   
    swapoff -a 
#   yum install -y yum-utils device-mapper-persistent-data lvm2 yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
#	yum -y install docker-ce
	yum -y install docker
	systemctl enable docker && systemctl start docker
    cp -f /vagrant/kube.repo /etc/yum.repos.d/kubernetes.repo
	# Set SELinux in permissive mode (effectively disabling it)
	setenforce 0
	sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
	yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
	systemctl enable kubelet && systemctl start kubelet
	cp -f /vagrant/k8s.conf /etc/sysctl.d/k8s.conf
	sysctl --system
	sysctl net.bridge.bridge-nf-call-iptables=1
	kubeadm init --apiserver-advertise-address=172.28.128.5 --pod-network-cidr=10.244.0.0/16
	mkdir -p $HOME/.kube
	cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	chown $(id -u):$(id -g) $HOME/.kube/config
#	export KUBECONFIG=/etc/kubernetes/admin.conf
	kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
	kubectl taint nodes --all node-role.kubernetes.io/master-
	curl https://raw.githubusercontent.com/helm/helm/master/scripts/get > get_helm.sh
	export PATH=$PATH:/usr/local/bin
	echo "export PATH=$PATH:/usr/local/bin" >> $HOME/.bashrc
	echo "export PATH=$PATH:/usr/local/bin" >> $HOME/.bash_profile
	source $HOME/.bashrc
	chmod 700 get_helm.sh
    ./get_helm.sh
	kubectl create serviceaccount --namespace kube-system tiller
	kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
	helm  init --service-account tiller
	sleep 100
	kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'  
#	mkdir /mnt/my_pv_data
#	kubectl create -f /vagrant/my_pv.yml
#	kubectl create -f /vagrant/my_pv_claim.yml
	mkdir /mnt/p1
	kubectl apply -f /vagrant/sc.yml
	kubectl apply -f /vagrant/pv.yml

	kubeadm token create --print-join-command
	#helm install --name my-jenkins --set Persistence.ExistingClaim=my-pv-claim stable/jenkins
#	sleep 100
	yum -y install httpd
	cp -f /vagrant/httpd.conf /etc/httpd/conf/httpd.conf
	systemctl enable httpd && systemctl start httpd
	#nohup kubectl port-forward service/my-jenkins 80:8080 > /dev/null &
	
	su vagrant
	
	kubectl create namespace namespace-kube
	openssl genrsa -out user_kube.key 2048
	openssl req -new -key user_kube.key -out user_kube.csr -subj "/CN=user_kube/O=group_kube"
	sudo openssl x509 -req -in user_kube.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out user_kube.crt -days 500
	mkdir .kube
	cp /vagrant/kube_config.yml /home/vagrant/kube_config
	kubectl config --kubeconfig=kube_config set-cluster kubernetes --server=https://172.28.128.5:6443 --certificate-authority=$HOME/.kube/ca.crt
	kubectl config --kubeconfig=kube_config set-credentials user_kube --client-certificate=user_kube.crt --client-key=user_kube.key
	kubectl config --kubeconfig=kube_config set-context kube_context --cluster=kubernetes --namespace=kube --user=user_kube
	kubectl config --kubeconfig=kube_config use-context kube_context
	cp /home/vagrant/kube_config /home/vagrant/.kube/config
	cp /home/vagrant/user* /home/vagrant/.kube/
	sudo cp /etc/kubernetes/pki/ca.crt /home/vagrant/.kube/
	
	sudo su
	
	kubectl create clusterrolebinding cluster_binding_kube --clusterrole=cluster-admin --user=user_kube
	helm install --name prometheus-operator stable/prometheus-operator --namespace monitoring
	helm repo add jfrog https://charts.jfrog.io
	helm repo update
#	helm install --name artifactory --set nginx.persistence.size=2Gi --set artifactory.persistence.enabled=false --set postgresql.persistence.enabled=false --set postgresql.postgresPassword=12_hX34qwerQ2 jfrog/artifactory
	#helm repo add coreos https://s3-eu-west-1.amazonaws.com/coreos-charts/stable/
	#helm install --name prometheus-operator coreos/prometheus-operator --namespace monitoring
	#helm install --name prometheus coreos/prometheus -f /vagrant/prometheus_values.yaml --namespace monitoring
	#helm install --name alertmanager coreos/alertmanager --namespace monitoring
	#helm install --name grafana coreos/grafana --namespace monitoring
#	cd /vagrant/exporters
#	for i in exporter-kube-controller-manager exporter-kube-dns exporter-kube-etcd exporter-kubelets exporter-kubernetes exporter-kube-scheduler exporter-kube-state exporter-node
#		do echo $i
#		helm install coreos/$i --name $i --namespace monitoring -f $i/values.yaml
#	done
	sleep 50
	prom_ip=`kubectl describe pod prometheus-prometheus-operator-prometheus --namespace=monitoring | grep -m1 IP | awk '{print $2}'`
	graf_ip=`kubectl describe pod prometheus-operator-grafana --namespace monitoring | grep -m1 IP | awk '{print $2}'`
	art_ip=`kubectl describe pod artifactory-artifactory-0 | grep -m1 IP | awk '{print $2}'`
	
	#Add ip to httpd conf to proxy and access via http://172.28.128.5:8090/targets
	sed -i -E "s/(ProxyPass[^0-9]*)([0-9]*\.[0-9]*\.[0-9]*\.[0-9]*)(:9090.*)/\1$prom_ip\3/" /etc/httpd/conf/httpd.conf
	sed -i -E "s/(ProxyPass[^0-9]*)([0-9]*\.[0-9]*\.[0-9]*\.[0-9]*)(:3000.*)/\1$graf_ip\3/" /etc/httpd/conf/httpd.conf
	sed -i -E "s/(ProxyPass[^0-9]*)([0-9]*\.[0-9]*\.[0-9]*\.[0-9]*)(:8081.*)/\1$art_ip\3/" /etc/httpd/conf/httpd.conf
	systemctl restart httpd
	kubectl get secrets prometheus-operator-grafana --namespace monitoring -o yaml | grep -m1 admin-password | awk '{print $2}' | base64 --decode
	
	#Clenup:
	#helm delete `helm list | awk '{if ($1!="my-jenkins" && $1!="NAME") {print $1}}'`
    #helm delete --purge `helm ls --deleted | awk '{if ($1!="my-jenkins" && $1!="NAME") {print $1}}'`
	#kubectl delete crd prometheuses.monitoring.coreos.com
	#kubectl delete crd prometheusrules.monitoring.coreos.com
	#kubectl delete crd servicemonitors.monitoring.coreos.com
	#kubectl delete crd alertmanagers.monitoring.coreos.com
	#helm del --purge artifactory
   EOF
end

(1..2).each do |i|
  config.vm.define "node#{i}" do |node|
    node.vm.network "private_network", ip: "172.28.128.#{i+1}"
	node.vm.provider "virtualbox" do |vb|
	vb.memory = "1024"
	vb.name="KubeNode#{i}"
#	vb.gui = true
	end
   node.vm.hostname="node#{i}"
   node.vm.provision 'shell', inline: <<-EOF
    sudo swapoff -a 
#   sudo yum install -y yum-utils device-mapper-persistent-data lvm2 sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
#	sudo yum -y install docker-ce
	sudo yum -y install docker
	sudo systemctl enable docker && systemctl start docker
    sudo cp -f /vagrant/kube.repo /etc/yum.repos.d/kubernetes.repo
	# Set SELinux in permissive mode (effectively disabling it)
	sudo setenforce 0
	sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
	sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
	sudo systemctl enable kubelet && systemctl start kubelet
	sudo cp -f /vagrant/k8s.conf /etc/sysctl.d/k8s.conf
	sudo sysctl --system
	sudo sysctl net.bridge.bridge-nf-call-iptables=1
	sudo curl -O https://copr.fedorainfracloud.org/coprs/ibotty/prometheus-exporters/repo/epel-7/ibotty-prometheus-exporters-epel-7.repo
	sudo mv ibotty-prometheus-exporters-epel-7.repo /etc/yum.repos.d/_copr_ibotty-prometheus-exporters.repo
	sudo yum -y install node_exporter
	sudo systemctl start node_exporter
   EOF
   end 
end
end
	
  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # NOTE: This will enable public access to the opened port
  # config.vm.network "forwarded_port", guest: 80, host: 8080

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine and only allow access
  # via 127.0.0.1 to disable public access
  # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  # config.vm.network "private_network", ip: "192.168.33.10"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  # config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end
  #
  # View the documentation for the provider you are using for more
  # information on available options.

  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.
  # config.vm.provision "shell", inline: <<-SHELL
  #   apt-get update
  #   apt-get install -y apache2
  # SHELL




