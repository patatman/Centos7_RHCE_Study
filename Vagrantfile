# -*- mode: ruby -*-
# vi: set ft=ruby :
#

Vagrant.configure(2) do |config|
  # 
  # Vagrant boxes for libvirt or virtualbox
  # 
  #Change to bento/centos-7.2 if you didn't build the box with packer
  #If you didn't build with packer then some settings below will need to be changed
  #  packer build packer.json
  config.vm.box = "centos7.2"
  #comment below line if you didn't build with packer
  config.vm.box_url = "./centos7.2.box"
  config.vm.provider "virtualbox" do |vb|
    vb.customize ["modifyvm", :id, "--memory", "1024"]
  config.vm.synced_folder ".", "/vagrant", disabled: 'true'
  config.vm.synced_folder ".", "/vagrant/rhce", create: 'true'
  end
#
# 
#
  config.vm.define :ipa do |ipa|
    ipa.vm.network "private_network", ip: "192.168.123.230"
    ipa.vm.hostname = "ipa.rhce.lab"
    ipa.vm.synced_folder "./ipa", "/vagrant/ipa"
    ipa.vm.provision "shell", inline: $common
    ipa.vm.provision "shell", inline: $ipa
  end

  config.vm.define :server1 do |server1|
    server1.vm.network "private_network", ip: "192.168.123.210"
    server1.vm.network "private_network", ip: "192.168.123.211",virtualbox__intnet: "true",nic_type: "virtio"
    server1.vm.network "private_network", ip: "192.168.123.212",virtualbox__intnet: "true",nic_type: "virtio"
    server1.vm.network "private_network", ip: "192.168.123.213",virtualbox__intnet: "true",nic_type: "virtio"
    server1.vm.hostname = "server1.rhce.lab"
    server1.vm.provision "shell", inline: $common
    server1.vm.provision "shell", inline: $server1
    server1.vm.synced_folder "rhce/", "/var/www/html/rhce", 
			owner: "apache", 
			group: "apache", 
			create: "true"
    server1.vm.synced_folder "conf.d/", "/etc/httpd/conf.d"
    server1.vm.provider "virtualbox" do |vb|
		unless File.exist?('server1-drive1.vmdk')
			vb.customize ['createhd','--filename','server1-drive1.vmdk','--variant','Fixed','--size', 1 * 1024]
      		end
      vb.customize ['storageattach', :id, '--storagectl', 'IDE Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', 'server1-drive1.vmdk']
  end
end
  config.vm.define :server2 do |server2|
    server2.vm.network "private_network", ip: "192.168.123.220"
    server2.vm.network "private_network", ip: "192.168.123.221",virtualbox__intnet: "true",nic_type: "virtio"
    server2.vm.network "private_network", ip: "192.168.123.222",virtualbox__intnet: "true",nic_type: "virtio"
    server2.vm.network "private_network", ip: "192.168.123.223",virtualbox__intnet: "true",nic_type: "virtio"
    server2.vm.hostname = "server2.rhce.lab"
    server2.vm.provision "shell", inline: $common
    server2.vm.provision "shell", inline: $server2
    server2.vm.provider "virtualbox" do |vb|
      		unless File.exist?('server2-drive1.vmdk')
			vb.customize ['createhd', '--filename', 'server2-drive1.vmdk', '--variant', 'Fixed', '--size', 1 * 1024]
      		end
        vb.customize ['storageattach', :id, '--storagectl', 'IDE Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', 'server2-drive1.vmdk']
  end
end
#
# Common node provisioning
#
$common = <<SCRIPT
#!/bin/sh
>&2 echo Common setup
#
# Check internet connection
#
ping -c 2 -W 2 google-public-dns-a.google.com
if [[ $? != 0 ]]
then
  echo "Can't connect to internet" >&2
  exit 1
fi
#
# Turning off firewalld (clean iptables rules)
#
systemctl stop firewalld.service
systemctl disable firewalld.service
#
# Turn on our interfaces, wich may not be started by vagrant
#
systemctl restart NetworkManager.service
systemctl stop network.service
systemctl start network.service
chkconfig network on
#Disable since epel isn't for sure available on exam
#yum -y --disableplugin=fastestmirror install epel-release
#
SCRIPT
#
# ipa node provisioning
#
$ipa = <<SCRIPT
#!/bin/sh
>&2 echo Ipa setup
#
echo > /etc/hosts
echo -e "192.168.123.230 ipa.rhce.lab ipa\n" >> /etc/hosts
echo -e "192.168.123.220 server2.rhce.lab server2\n" >> /etc/hosts
echo -e "192.168.123.210 server1.rhce.lab server1\n" >> /etc/hosts
systemctl restart NetworkManager
#Removing update
#yum -y --disableplugin=fastestmirror update
hostnamectl set-hostname ipa.rhce.lab
yum -y --disableplugin=fastestmirror install ipa-server bind-dyndb-ldap
systemctl isolate multi-user.target
#check in ipa folder for setup commands
systemctl enable firewalld
systemctl start firewalld
for i in http https ldap ldaps kerberos kpasswd dns ntp; do firewall-cmd --permanent --add-service $i; done
firewall-cmd --reload
ipa-server-install --realm=RHCE.LAB --domain=rhce.lab --ds-password=password --master-password=password --admin-password=password --mkhomedir --hostname=ipa.rhce.lab --ip-address=192.168.123.230 -U
echo "password" | kinit admin
klist
echo "password" | ipa user-add lisa --first=lisa --last=jones --password
echo "password" | ipa user-add linda --first=linda --last=thomsen --password
yum install -y vsftpd
systemctl enable vsftpd
systemctl start vsftpd
cp ~/cacert.p12 /var/ftp/pub
firewall-cmd --permanent --add-service ftp
firewall-cmd --reload
ipa host-add --force server1.rhce.lab
ipa host-add --force server2.rhce.lab
ipa service-add --force nfs/server1.rhce.lab
ipa service-add --force nfs/server2.rhce.lab
ipa service-add --force cifs/server1.rhce.lab
ipa service-add --force cifs/server2.rhce.lab
SCRIPT
#
# server1 node provisioning
#
$server1 = <<SCRIPT
#!/bin/sh
>&2 echo Server1 setup
echo > /etc/hosts
echo -e "192.168.123.220 server2.rhce.lab server2\n" >> /etc/hosts
echo -e "192.168.123.210 server1.rhce.lab server1\n" >> /etc/hosts
echo -e "192.168.123.230 ipa.rhce.lab ipa\n" >> /etc/hosts
systemctl restart NetworkManager
systemctl enable firewalld.service
systemctl start firewalld.service
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
useradd -s /sbin/nologin suser0
useradd -s /sbin/nologin suser1
useradd -s /sbin/nologin suser2
useradd -s /sbin/nologin tuser0
useradd -s /sbin/nologin tuser1
useradd -s /sbin/nologin tuser2
yum -y --disableplugin=fastestmirror install ipa-client
semodule -i /vagrant/rhce/selinux/vmblock.pp
semodule -R
systemctl restart httpd
systemctl enable httpd
ipa-client-install --realm=RHCE.LAB --domain=rhce.lab --server=ipa.rhce.lab --mkhomedir --hostname server1.rhce.lab --ip-address=192.168.123.210 -p admin --password=password -U
echo "password" | kinit admin
klist
ipa-getkeytab -s ipa.rhce.lab -p nfs/server1.rhce.lab -k /etc/krb5.keytab
kinit -k nfs/server1.rhce.lab
klist
if [ ! -d '/repo' ];then
	mkdir /repo
fi
SCRIPT
#
# server2 node provisioning
#
$server2 = <<SCRIPT
#!/bin/sh
>&2 echo Server2 setup
echo > /etc/hosts
echo -e "192.168.123.220 server2.rhce.lab server2\n" >> /etc/hosts
echo -e "192.168.123.210 server1.rhce.lab server1\n" >> /etc/hosts
echo -e "192.168.123.230 ipa.rhce.lab ipa\n" >> /etc/hosts
systemctl restart NetworkManager
systemctl enable firewalld.service
systemctl start firewalld.service
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
useradd -s /sbin/nologin suser0
useradd -s /sbin/nologin suser1
useradd -s /sbin/nologin suser2
useradd -s /sbin/nologin tuser0
useradd -s /sbin/nologin tuser1
useradd -s /sbin/nologin tuser2
yum -y --disableplugin=fastestmirror install ipa-client
ipa-client-install --realm=RHCE.LAB --domain=rhce.lab --server=ipa.rhce.lab --mkhomedir --hostname server2.rhce.lab --ip-address=192.168.123.220 -p admin --password=password -U
echo "password" | kinit admin
ipa-getkeytab -s ipa.rhce.lab -p nfs/server2.rhce.lab -k /etc/krb5.keytab
kinit -k nfs/server2.rhce.lab
klist
SCRIPT
end
