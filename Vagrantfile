# -*- mode: ruby -*-
# vi: set ft=ruby :

NODES = {
  "manager1" => "192.168.99.100",
  "worker1"  => "192.168.99.101",
  "worker2"  => "192.168.99.102",
}

Vagrant.configure("2") do |config|
  NODES.each do |node_name, ip_address|
    config.vm.define node_name do |node|
      node.vm.box = "bento/ubuntu-24.04"
      node.vm.hostname = node_name
      node.vm.network "private_network", ip: ip_address

      node.vm.provider "virtualbox" do |vb|
        vb.name = node_name
        vb.memory = 1024
        vb.cpus = 1
      end

      node.vm.provision "shell", inline: <<-SHELL
        set -e

        # Ajout propre dans /etc/hosts sans doublons
        #{NODES.map { |name, ip| "grep -q '^#{ip} #{name}$' /etc/hosts || echo '#{ip} #{name}' | sudo tee -a /etc/hosts" }.join("\n")}

        # Installer Docker
        curl -fsSL https://get.docker.com -o get-docker.sh
        sudo sh get-docker.sh
        rm -f get-docker.sh

        # Ajouter vagrant au groupe docker
        sudo usermod -aG docker vagrant
      SHELL
    end
  end

  config.vm.define "manager1" do |manager|
    manager.vm.provision "shell", run: "always", inline: <<-SHELL
      set -e

      if ! sudo docker info 2>/dev/null | grep -q "Swarm: active"; then
        sudo docker swarm init --advertise-addr 192.168.99.100
        sudo docker swarm join-token worker -q > /vagrant/worker-token
      fi
    SHELL
  end

  ["worker1", "worker2"].each do |worker|
    config.vm.define worker do |w|
      w.vm.provision "shell", run: "always", inline: <<-SHELL
        set -e

        if [ -f /vagrant/worker-token ]; then
          if ! sudo docker info 2>/dev/null | grep -q "Swarm: active"; then
            token=$(cat /vagrant/worker-token)
            sudo docker swarm join --token $token 192.168.99.100:2377
          fi
        else
          echo "worker-token introuvable pour l'instant, le manager n'a peut-être pas fini son init."
        fi
      SHELL
    end
  end
end