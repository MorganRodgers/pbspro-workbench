# -*- mode: ruby -*-
# vi: set ft=ruby :

def install_deps!(node)
  node.vm.provision "shell", inline: "cp -f /vagrant/hosts /etc/hosts"
  deps = [
    'expat',
    'hwloc',
    'libedit',
    'libical',
    'libICE',
    'libSM',
    'perl-Env',
    'perl-Switch',
    'postgresql-contrib',
    'postgresql-server',
    'python',
    'sendmail',
    'sudo',
    'tcl',
    'tk',
    'unzip',

    'the_silver_searcher', # *
    'vim',  # *
    'zip'

    # * purely for convenience
  ]
  node.vm.provision "shell", inline: "yum install -y #{deps.join(' ')}"
end

def add_ood_user!(node)
  node.vm.provision "shell", inline: <<-SHELL
    groupadd ood
    useradd --create-home --gid ood ood
    echo -n "ood" | passwd --stdin ood
  SHELL
end


Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"
  config.vm.synced_folder ".", "/vagrant", type: "sshfs"
  config.vm.synced_folder "./ood-home", "/home/ood", type: "virtualbox", mount_options: ["uid=1001","gid=1001"]
  config.ssh.forward_x11 = true

  config.vm.provider "virtualbox" do |v|
    v.memory = 1024
    v.cpus = 2
  end

  # Client / OOD node
  config.vm.define "ood", primary: true, autostart: true do |ood|
    # ood.vm.network "forwarded_port", guest: 80, host: 8080
    ood.vm.network "private_network", ip: "10.0.0.100"
    ood.vm.provision "shell", inline: "hostnamectl set-hostname ood"
    install_deps!(ood)
    ood.vm.provision "shell", inline: <<-SHELL
      if [[ ! -e /vagrant/pbspro_19.1.1.centos7.zip ]]; then
        (
          cd /vagrant || exit 1
          curl -L https://github.com/PBSPro/pbspro/releases/download/v19.1.1/pbspro_19.1.1.centos7.zip -o pbspro_19.1.1.centos7.zip
          unzip pbspro_19.1.1.centos7.zip
        )
      fi
    SHELL
    ood.vm.provision "shell", inline: "(cd /vagrant/pbspro_19.1.1.centos7 && rpm -i pbspro-client-19.1.1-0.x86_64.rpm)"
    ood.vm.provision "shell", inline: "perl -pi -e 's[CHANGE_THIS_TO_PBS_PRO_SERVER_HOSTNAME][head]' /etc/pbs.conf"
  end

  # Compute node
  config.vm.define "pbscompute", primary: false, autostart: true do |pbscompute|
    pbscompute.vm.network "private_network", ip: "10.0.0.102"
    pbscompute.vm.provision "shell", inline: "hostnamectl set-hostname pbscompute"
    install_deps!(pbscompute)
    pbscompute.vm.provision "shell", inline: <<-SHELL
      (cd /vagrant/pbspro_19.1.1.centos7 && rpm -i pbspro-execution-19.1.1-0.x86_64.rpm)
      perl -pi -e 's[CHANGE_THIS_TO_PBS_PRO_SERVER_HOSTNAME][head]' /etc/pbs.conf
      perl -pi -e 's[CHANGE_THIS_TO_PBS_PRO_SERVER_HOSTNAME][head]' /var/spool/pbs/mom_priv/config
      yum install -y Lmod openmpi
      systemctl enable pbs --now
    SHELL
  end
  
  # Server Head Node
  config.vm.define "head", primary: false, autostart: true do |head|
    head.vm.network "private_network", ip: "10.0.0.101"
    head.vm.provision "shell", inline: "hostnamectl set-hostname head"
    install_deps!(head)
    head.vm.provision "shell", inline: <<-SHELL
      (cd /vagrant/pbspro_19.1.1.centos7 && rpm -i pbspro-server-19.1.1-0.x86_64.rpm)
      systemctl enable pbs --now
      export PATH="$PATH:/opt/pbs/bin"
      qmgr -c 'create node pbscompute'
      # qmgr -c 'create node head'  # use the head node as a compute node
      qmgr -c 'set server acl_roots+=ood'
      qmgr -c 'set server flatuid=true'
    SHELL
  end
end

