# -*- mode: ruby -*-
# vi: set ft=ruby :

#Windows Vars
vmname = "dc"
hostname = "dc"
domain_fqdn = "example.local"
domain_netbios = "EXAMPLE"
domain_safemode_password = "Admin123#"

#Linux Vars
IMAGE_NAME = "geerlingguy/rockylinux8"

Vagrant.configure("2") do |config|
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus = 2
  end

  config.vm.define "dc" do |windows|
    windows.vm.box = "gusztavvargadr/windows-server"
    windows.vm.hostname = hostname

    # use the plaintext WinRM transport and force it to use basic authentication.
    # NB this is needed because the default negotiate transport stops working
    #    after the domain controller is installed.
    #    see https://groups.google.com/forum/#!topic/vagrant-up/sZantuCM0q4
    windows.winrm.transport = :plaintext
    windows.winrm.basic_auth_only = true

    windows.vm.communicator = "winrm"
    windows.vm.network :forwarded_port, guest: 5985, host: 5985, id: "winrm", auto_correct: true
    windows.vm.network :forwarded_port, guest: 22, host: 2222, id: "ssh", auto_correct: true
    windows.vm.network :forwarded_port, guest: 3389, host: 3389, id: "rdp", auto_correct: true
    windows.vm.network :private_network, ip: "192.168.56.10"

    windows.vm.network :forwarded_port, guest: 389, host: 7389, id: "ldap", auto_correct: true
    windows.vm.network :forwarded_port, guest: 636, host: 7636, id: "ldaps", auto_correct: true

    windows.vm.provision "shell", path: "scripts/remove_defender_core.ps1", privileged: false
    windows.vm.provision "shell", path: "scripts/disable_wu.ps1", privileged: false
    windows.vm.provision "shell", path: "scripts/disable_rdp_nla.ps1", privileged: false
    windows.vm.provision "shell", path: "scripts/install_ad.ps1", privileged: false
    windows.vm.provision "shell", reboot: true
    windows.vm.provision "shell", path: "scripts/configure_ad.ps1", privileged: false, args: "'#{domain_fqdn}' '#{domain_netbios}' '#{domain_safemode_password}'"
    windows.vm.provision "shell", path: "scripts/certificate.ps1", privileged: false, args: "'#{hostname}' '#{domain_fqdn}'"
    windows.vm.provision "shell", reboot: true
    windows.vm.provision "shell", path: "scripts/install_adexplorer.ps1", privileged: false
    windows.vm.provision "shell", path: "scripts/importusers.ps1", privileged: false, args: "'#{domain_fqdn}'"

    windows.vm.provider "virtualbox" do |vb, override|
      vb.name = vmname
      vb.gui = false
      vb.customize ["modifyvm", :id, "--memory", 1024]
      vb.customize ["modifyvm", :id, "--cpus", 1]
      vb.customize ["modifyvm", :id, "--vram", "16"]
      vb.customize ["modifyvm", :id, "--clipboard", "bidirectional"]
      vb.customize ["setextradata", "global", "GUI/SuppressMessages", "all" ]
      vb.customize ["modifyvm", :id, "--macaddress1", "auto"]
      vb.customize ["modifyvm", :id, "--macaddress2", "auto"]
    end
  end

  config.vm.define "idm-server" do |master|
    master.vm.box = IMAGE_NAME
    master.vm.synced_folder '.', '/vagrant', disabled: true
    master.vm.hostname = "idm-server.example.local"
    master.vm.network "private_network", ip: "192.168.56.20"
    master.vm.network "forwarded_port", guest: 80, host: 8080
    master.vm.network "forwarded_port", guest: 443, host: 8443
    master.vm.provision "ansible" do |ansible|
      ansible.groups = {
        "ipaservers" => ["idm-server"],
        "ipaservers:vars" => {"ipaadmin_password" => "redhat123",
                              "ipadm_password" => "redhat123",
                              "ipaserver_domain" => "example.local",
                              "ipaserver_realm" => "EXAMPLE.LOCAL",
                              "ipaserver_setup_dns" => "no",
                              "ipaserver_setup_adtrust" => "yes",
                              "ipaserver_netbios_name" => "EXAMPLE",
                              "ipaserver_auto_forwarders" => "no",
                              "ipaserver_install_packages" => "yes"
                             }
      }
      ansible.galaxy_role_file = "provisioning/requirements.yml"
      ansible.playbook = "provisioning/preparation-server.yml"
      ansible.verbose = "-vvv"
    end
  end

  config.vm.define "idm-node1" do |node|
    node.vm.box = IMAGE_NAME
    node.vm.synced_folder '.', '/vagrant', disabled: true
    node.vm.hostname = "idm-node1.example.local"
    node.vm.network "private_network", ip: "192.168.56.21"
    node.vm.provision "ansible" do |ansible|
      ansible.groups = {
        "ipaservers" => ["idm-server"],
        "ipaclients" => ["idm-node1"],
        "ipaclients:vars" => {"ipaclient_domain" => "example.local",
                              "ipaadmin_principal" => "admin",
                              "ipaadmin_password" => "redhat123",
                              "ipaclient_servers" => "idm-server.example.local"
                             }
      }
      ansible.galaxy_role_file = "provisioning/requirements.yml"
      ansible.playbook = "provisioning/preparation-client.yml"
    end
  end

end