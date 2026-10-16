# -*- mode: ruby -*-
# vi: set ft=ruby :

MACHINES = {
  "backup_server" => { ip: "192.168.11.160", hostname: "backup-server", backup_disk: true  },
  "client"        => { ip: "192.168.11.150", hostname: "client",        backup_disk: false },
}

BACKUP_DISK      = File.join(File.dirname(File.expand_path(__FILE__)), ".vagrant", "backup_disk.vdi")
BACKUP_DISK_SIZE = 2048 # MB

Vagrant.configure("2") do |config|
  config.vm.box              = "ubuntu/jammy64"
  config.vm.box_check_update = false
  config.vm.boot_timeout     = 600

  MACHINES.each do |name, cfg|
    config.vm.define name do |node|
      node.vm.hostname = cfg[:hostname]
      node.vm.network "private_network", ip: cfg[:ip]

      node.vm.provider "virtualbox" do |vb|
        vb.name   = name
        vb.memory = 1024
        vb.cpus   = 1

        # Отдельный диск 2 ГБ под /var/backup
        if cfg[:backup_disk]
          unless File.exist?(BACKUP_DISK)
            vb.customize ["createmedium", "disk", "--filename", BACKUP_DISK,
                          "--format", "VDI", "--size", BACKUP_DISK_SIZE]
          end
          vb.customize ["storageattach", :id, "--storagectl", "SCSI",
                        "--port", 2, "--device", 0, "--type", "hdd", "--medium", BACKUP_DISK]
        end
      end

      # Ansible запускается один раз, после поднятия последней ВМ, сразу на обе
      if name == "client"
        node.vm.provision "ansible" do |ansible|
          ansible.playbook           = "ansible/playbook.yml"
          ansible.limit              = "all"
          ansible.compatibility_mode = "2.0"
          ansible.groups = {
            "backup"  => ["backup_server"],
            "clients" => ["client"],
          }
        end
      end
    end
  end
end
