servers = [
    { :hostname => 'clnt',
      :ips => '192.168.31.202',
      :ram => 2048,
      :cpuz => 1
    },
    {   :hostname => 'bckp',
        :ips => '192.168.31.201',
        :ram => 2048,
        :cpuz => 2,
        :dfile => './sata1.vdi',
        :size => 2048
    }
]

Vagrant.configure(2) do |config|
    servers.each do |machine|
        config.vm.define machine[:hostname] do |node|
            node.vm.box = 'centos/7'
            node.vm.box_version = "2004.01"
            node.vm.hostname = machine[:hostname]
           
            node.vm.network :public_network, ip: machine[:ips], bridge: "wlp58s0"
            node.vm.provider "virtualbox" do |vb|
                vb.memory = machine[:ram]
                vb.cpus = machine[:cpuz]
                needsController = false
                if (!machine[:dfile].nil?)
                    unless File.exist?(machine[:dfile])
                        vb.customize ['createhd', '--filename', machine[:dfile], '--variant', 'Fixed', '--size', machine[:size]]
                        needsController =  true
                    end
                end
                if needsController == true
                    vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
                    vb.customize ["storageattach", :id, '--storagectl', 'SATA', "--port", 1, "--device", 0, "--type", "hdd", "--medium", machine[:dfile]]
                end
            end
            node.vm.provision "shell", inline: <<-SHELL
                mkdir -p ~root/.ssh; cp ~vagrant/.ssh/auth* ~root/.ssh
                sed -i '65s/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
                systemctl restart sshd
            SHELL
            node.vm.provision :ansible do |ansible|
                ansible.playbook = "playbooks/playbook-#{node.vm.hostname}.yml"
            end
#            if (machine[:hostname] == "clnt")
#                config.vm.provision "shell", path: "prepeare.sh"
#            end

        end
        
    end
end
