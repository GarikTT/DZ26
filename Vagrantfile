# -*- mode: ruby -*-
# vim: set ft=ruby :
home = ENV['HOME']
ENV["LC_ALL"] = "en_US.UTF-8"

MACHINES =
{
	:backup =>
	{
		:box_name => "centos/7",
		:ip_addr => '192.168.56.10',
    	:disks =>
		{
        	:sata1 =>
			{
            	:dfile => './sata1.vdi',
            	:size => 2048,
            	:port => 1
        	},
		}
	},
	:client =>
	{
        :box_name => "centos/7",
        :ip_addr => '192.168.56.15',
    	:disks =>
		{
    	}
  	},
}

Vagrant.configure("2") do |config|

    config.vm.box_check_update = false
    MACHINES.each do |boxname, boxconfig|
  
        config.vm.define boxname do |box|
  
            box.vm.box = boxconfig[:box_name]
            box.vm.host_name = boxname.to_s
  
            box.vm.network "private_network", ip: boxconfig[:ip_addr]
  
            box.vm.provider :virtualbox do |vb|
                    vb.customize ["modifyvm", :id, "--memory", "512"]
                    needsController = false
            boxconfig[:disks].each do |dname, dconf|
                unless File.exist?(dconf[:dfile])
                vb.customize ['createhd', '--filename', dconf[:dfile], '--variant', 'Fixed', '--size', dconf[:size]]
                needsController =  true
                end
  
            end
                    if needsController == true
                       vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
                       boxconfig[:disks].each do |dname, dconf|
                       vb.customize ['storageattach', :id,  '--storagectl', 'SATA', '--port', dconf[:port], '--device', 0, '--type', 'hdd', '--medium', dconf[:dfile]]
                       end
                    end
            end
  
        box.vm.provision "shell", inline: <<-SHELL
			#Разрешаем подключение пользователей по SSH с использованием пароля
			sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
			#Перезапуск службы SSHD
			service sshd restart
        	# Добавлено для создания файловой системы и монтирования диска
            yum update
			yum install -y mc
			yum install epel-release -y
            yum install borgbackup -y
        	if [ -e /dev/sdb ]; then
        		useradd borg ; echo -e "garik123\ngarik123" | passwd borg
          		mkfs.ext4 -F /dev/sdb
          		mkdir /var/backup
				chmod 700 /var/backup
				mount /dev/sdb /var/backup
          		echo "/dev/sdb    /var/backup    ext4    defaults    0    0" >> /etc/fstab
          		echo "borg ALL=(ALL)  ALL" >> /etc/sudoers
				chown borg:borg /var/backup/
				lsblk
				#echo borg ALL=(ALL)  ALL > /etc/sudoers
        	fi
        SHELL
  
        end
    end
  end
