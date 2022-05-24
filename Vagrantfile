IMAGE_NAME_UBUNTU = "bento/ubuntu-18.04"
IMAGE_NAME_WINDOWS = "gusztavvargadr/windows-10"
IMAGE_NAME_WINDOWS_SERVER = "gusztavvargadr/windows-server"

MASTER_NUM = 1
MASTER_CPU = 3
MASTER_MEM = 3024

UBUNTU_NUM = 1
UBUNTU_CPU = 3
UBUNTU_MEM = 4072

WINDOWS_NUM = 1
WINDOWS_CPU = 2
WINDOWS_MEM = 3000

PRIVATE_IP_BASE = "10.0.2."
PUBLIC_IP_BASE = "192.168.1."

VAGRANT_DISABLE_VBOXSYMLINKCREATE=1

$script = <<-SCRIPT
echo Start provisioning...
USER=vagrant
sudo apt-get update
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    git \
    vim \
    openjdk-11-jdk
echo 'export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64' >> ~/.bashrc
echo 'export PATH=$PATH:$JAVA_HOME/bin' >> ~/.bashrc
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo  tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo docker --version
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker–compose –version
sudo usermod -aG docker $USER
newgrp docker
SCRIPT

$script2= <<-SCRIPT
JFROG_HOME=/home/vagrant/jfrog
mkdir -p $JFROG_HOME/artifactory/var/etc/
cd $JFROG_HOME/artifactory/var/etc/
touch ./system.yaml
sudo chown -R 1030:1030 $JFROG_HOME/artifactory/var
# docker run --name artifactory -v $JFROG_HOME/artifactory/var/:/var/opt/jfrog/artifactory -d -p 8081:8081 -p 8082:8082 releases-docker.jfrog.io/jfrog/artifactory-oss:latest
SCRIPT

Vagrant.configure("2") do |config|
    #config.ssh.insert_key = false

    (1..MASTER_NUM).each do |i|
        config.vm.define "controller-ubuntu-#{i}" do |controller|
          controller.vm.box = IMAGE_NAME_UBUNTU
#           controller.vm.network :private_network, ip: "#{PRIVATE_IP_BASE}#{i*5 + 10}"
          controller.vm.network :private_network, type: "dhcp"
          controller.vm.network :forwarded_port, guest: 8080, host: 8080, host_ip: "127.0.0.1"
          controller.vm.network :forwarded_port, guest: 8081, host: 8081, host_ip: "127.0.0.1"
          controller.vm.network :forwarded_port, guest: 8082, host: 8082, host_ip: "127.0.0.1"
          controller.vm.network :forwarded_port, guest: 50000, host: 50000, host_ip: "127.0.0.1"
# 	      controller.vm.network "public_network", ip: "#{PUBLIC_IP_BASE}#{i*5 + 10}", bridge: [
#                                                "Intel(R) Ethernet Connection (7) I219-V",
#                                                "Realtek PCIe GbE Family Controller"
#                                            ]
          controller.vm.hostname = "controller-ubuntu-#{i}"
          controller.vm.provider "virtualbox" do |vb|
            vb.memory = MASTER_MEM
            vb.cpus = MASTER_CPU
          end
          controller.vm.provision "shell", inline: $script
          controller.vm.provision "shell", inline: $script2
        end
    end

    (1..UBUNTU_NUM).each do |i|
        config.vm.define "slave-ubuntu-#{i}" do |slave|
          slave.vm.box = IMAGE_NAME_UBUNTU
          controller.vm.network :private_network, type: "dhcp"
#           slave.vm.network "private_network", ip: "#{PRIVATE_IP_BASE}#{i*5 + 100}"
# 	      slave.vm.network "public_network", ip: "#{PUBLIC_IP_BASE}#{i*5 + 100}", bridge: [
#                                                "Intel(R) Ethernet Connection (7) I219-V",
#                                                "Realtek PCIe GbE Family Controller"
#                                            ]
          slave.vm.hostname = "slave-ubuntu-#{i}"
          slave.vm.provider "virtualbox" do |vb|
            vb.memory = UBUNTU_MEM
            vb.cpus = UBUNTU_CPU
          end
          slave.vm.provision "shell", inline: $script
        end
    end

#     (1..WINDOWS_NUM).each do |i|
#                 config.vm.define "agent-windows-server#{i}" do |agent|
#                   agent.vm.box = IMAGE_NAME_WINDOWS_SERVER
#                   agent.vm.network "private_network", ip: "#{PRIVATE_IP_BASE}#{i*5 + 150}"
#                   agent.vm.network "public_network", ip: "#{PUBLIC_IP_BASE}#{i*5 + 150}", bridge: [
#                                                        "Intel(R) Ethernet Connection (7) I219-V",
                                                         "Realtek PCIe GbE Family Controller"
#                                                    ]
#                   agent.vm.hostname = "agent-windows-server-#{i}"
#                   agent.vm.provider "virtualbox" do |vb|
#                     vb.memory = WINDOWS_MEM
#                     vb.cpus = WINDOWS_CPU
#                   end
#                 end
#             end

end
