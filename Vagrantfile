VAGRANTFILE_API_VERSION = "2"
VAGRANT_DISABLE_VBOXSYMLINKCREATE = "1"

# Definição dos arquivos de disco
disk_files = {
  'node1' => './disk-0-1.vdi',
  'node2' => './disk-0-2.vdi', 
  'node3' => './disk-0-3.vdi',
  'node4' => './disk-0-4.vdi',
  'repo'  => './disk-1-3.vdi'
}

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # Configurações globais
  config.ssh.insert_key = false
  config.vm.box_check_update = false
  config.vm.box = "bento/almalinux-9"

  # Repo Server
  config.vm.define "repo" do |repo|
    repo.vm.network "private_network", ip: "192.168.55.199"
    repo.vm.synced_folder ".", "/vagrant", type: "rsync", rsync__exclude: ".git/"
    
    repo.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.name = "repo"
      
      # Criação do disco adicional
      disk_file = disk_files['repo']
      unless File.exist?(disk_file)
        vb.customize ['createhd', '--filename', disk_file, '--variant', 'Standard', '--size', 4 * 1024]
        # Anexar o disco ao controlador SATA existente
        vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', disk_file]
      end
      # Script para formatação do disco
      repo.vm.provision "shell", inline: <<-SHELL
        # Aguardar o disco ficar disponível
        echo "Aguardando disco ficar disponível..."
        timeout=30
        while [ $timeout -gt 0 ] && [ ! -b /dev/sdb ]; do
          sleep 1
          timeout=$((timeout - 1))
        done
        
        if [ -b /dev/sdb ]; then
          echo "Disco /dev/sdb encontrado. Formatando..."
          
          # Verificar se já está formatado
          if ! blkid /dev/sdb >/dev/null 2>&1; then
            echo "Criando sistema de arquivos ext4 em /dev/sdb"
            sudo mkfs.ext4 -F /dev/sdb
            echo "Formatação concluída"
          else
            echo "Disco já formatado"
          fi
          
          # Criar ponto de montagem e montar
          sudo mkdir -p /mnt/data
          if ! mountpoint -q /mnt/data; then
            sudo mount /dev/sdb /mnt/data
            echo "Disco montado em /mnt/data"
          fi
          
          # Adicionar ao fstab para montagem automática
          if ! grep -q "/dev/sdb" /etc/fstab; then
            echo "/dev/sdb /mnt/data ext4 defaults 0 0" | sudo tee -a /etc/fstab
          fi
        else
          echo "ERRO: Disco /dev/sdb não encontrado após timeout"
          exit 1
        fi
      SHELL
    end

  end

  # Função para criar nós worker
  def create_worker_node(config, name, ip, memory, disk_file)
    config.vm.define name do |node|
      node.vm.network "private_network", ip: ip
      node.vm.synced_folder ".", "/vagrant", type: "rsync", rsync__exclude: [".git/", "*.vdi"]
      
      node.vm.provider "virtualbox" do |vb|
        vb.memory = memory
        vb.name = name
        
        # Criação do disco adicional
        unless File.exist?(disk_file)
          vb.customize ['createhd', '--filename', disk_file, '--variant', 'Fixed', '--size', 2 * 1024]
          # Anexar o disco ao controlador SATA existente
          vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', disk_file]
        end
      end

      # Script para formatação do disco
      node.vm.provision "shell", inline: <<-SHELL
        # Aguardar o disco ficar disponível
        echo "Aguardando disco ficar disponível..."
        timeout=30
        while [ $timeout -gt 0 ] && [ ! -b /dev/sdb ]; do
          sleep 1
          timeout=$((timeout - 1))
        done
        
        if [ -b /dev/sdb ]; then
          echo "Disco /dev/sdb encontrado. Formatando..."
          
          # Verificar se já está formatado
          if ! blkid /dev/sdb >/dev/null 2>&1; then
            echo "Criando sistema de arquivos ext4 em /dev/sdb"
            sudo mkfs.ext4 -F /dev/sdb
            echo "Formatação concluída"
          else
            echo "Disco já formatado"
          fi
          
          # Criar ponto de montagem e montar
          sudo mkdir -p /mnt/data
          if ! mountpoint -q /mnt/data; then
            sudo mount /dev/sdb /mnt/data
            echo "Disco montado em /mnt/data"
          fi
          
          # Adicionar ao fstab para montagem automática
          if ! grep -q "/dev/sdb" /etc/fstab; then
            echo "/dev/sdb /mnt/data ext4 defaults 0 0" | sudo tee -a /etc/fstab
          fi
        else
          echo "ERRO: Disco /dev/sdb não encontrado após timeout"
          exit 1
        fi
      SHELL
      end

   # Provisioning
   repo.vm.provision :shell, inline: <<-SHELL
     # Habilitar autenticação por senha SSH
     sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
     sudo systemctl restart sshd
     
     # Instalar repositórios e pacotes necessários
     sudo dnf config-manager --set-enabled crb
     sudo dnf install epel-release -y
     sudo dnf install -y sshpass python3-pip python3-devel httpd vsftpd createrepo ansible-core
     
     # Atualizar pip e instalar pacotes Python
     python3 -m pip install --user --upgrade pip pexpect
   SHELL    
  end

  # Criar nós worker
  create_worker_node(config, "node1", "192.168.55.201", "1024", disk_files['node1'])
  create_worker_node(config, "node2", "192.168.55.202", "1024", disk_files['node2'])
  create_worker_node(config, "node3", "192.168.55.203", "512", disk_files['node3'])
  create_worker_node(config, "node4", "192.168.55.204", "512", disk_files['node4'])

  # Control Node
  config.vm.define "control" do |control|
    control.vm.network "private_network", ip: "192.168.55.200"
    control.vm.synced_folder ".", "/vagrant", type: "rsync", rsync__exclude: [".git/", "*.vdi"]
    
    control.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
      vb.name = "control"
    end
    
    # Provisioning com Ansible
    control.vm.provision :ansible_local do |ansible|
      ansible.playbook = "/vagrant/playbooks/master.yml"
      ansible.install = false
      ansible.compatibility_mode = "2.0"
      ansible.inventory_path = "/vagrant/inventory"
      ansible.config_file = "/vagrant/ansible.cfg"
      ansible.limit = "all"
    end
  end
end