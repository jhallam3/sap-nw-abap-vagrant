# -*- mode: ruby -*-
# vi: set ft=ruby :

# Defaults and constants
vmNameFromEnv  = 'VAGRANT_SAPNW_VM_NAME'
argVagrantName = 'sapnw'
argMachineName = 'sap-nw752sp4'

unless ENV[vmNameFromEnv].to_s.strip.empty?
  argVagrantName = ENV[vmNameFromEnv]
  argMachineName = ENV[vmNameFromEnv]
  puts "[!] VAGRANT_SAPNW_VM_NAME env detected"
  puts "[!] VM name is redefined to: #{argVagrantName}"
end

# Create additional disk in VM directory
class VagrantPlugins::ProviderVirtualBox::Action::SetName
  alias_method :original_call, :call
  def call(env)
    machine = env[:machine]
    driver  = machine.provider.driver
    uuid    = driver.instance_eval { @uuid }
    ui      = env[:ui]

    # Find out folder of VM
    vm_folder = ""
    vm_info   = driver.execute("showvminfo", uuid, "--machinereadable")
    lines     = vm_info.split("\n")
    lines.each do |line|
      if line.start_with?("CfgFile")
        vm_folder = line.split("=")[1].gsub('"','')
        vm_folder = File.dirname(File.expand_path(vm_folder))
        ui.info "VM Folder is: #{vm_folder}"
        break
      end
    end

    disk_size = 60 * 1024  # 1GB
    disk_file = File.join(vm_folder, "sybase.vdi")
    ui.info "VM additional disk is: #{disk_file}"

    ui.info "Adding disk to VM ..."
    if File.exist?(disk_file)
      ui.info "  disk already exists"
    else
      ui.info "  creating new disk"
      driver.execute(
        'createhd', 
        '--filename', disk_file, 
        '--format',   'VDI', 
        '--size',     "#{disk_size}",
        '--variant', 'Standard'  # dynamic disk
      )
      ui.info "  attaching disk to VM"
      driver.execute(
        'storageattach', uuid,
        '--storagectl',  'SCSI',
        '--port',   '2',
        '--device', '0', 
        '--type',   'hdd',
        '--medium', disk_file
      )
    end

    original_call(env)
  end
end

# Vagrant config
Vagrant.configure("2") do |config|
  config.vm.box      = "ubuntu/xenial64"
  config.vm.hostname = "vhcalnplci"
  config.vm.define argVagrantName
  
  # Check for updates only on `vagrant box outdated`
  config.vm.box_check_update = false

  # stopsap may take time
  config.vm.graceful_halt_timeout = 600

  # Network
  config.vm.network "public_network"
 
  # Virtualbox settings
  config.vm.provider "virtualbox" do |vb|
    vb.name   = argMachineName
     vb.memory = "16384" # 16 GB
     vb.cpus = 4
   # vb.memory = "6144" # 6 GB
    # vb.memory = "4096" # 4 GB + enable add_swap.sh below !!!
  end

  # Provision scripts

#MAC FILE PERMISSIONS
config.vm.provision "shell", inline: <<-SHELL
    chmod +x /vagrant/scripts/provision/*.sh
    chmod +x /vagrant/distrib/install.sh
    
SHELL

  config.vm.provision "shell", path: "scripts/provision/add_disk.sh"
  # config.vm.provision "shell", path: "scripts/provision/add_swap.sh"
  config.vm.provision "shell", path: "scripts/provision/pre_install.sh"
  config.vm.provision "shell", path: "scripts/provision/install_nw.sh"
  config.vm.provision "shell", path: "scripts/provision/startup.sh"
  config.vm.provision "shell", path: "scripts/provision/post_install.sh"
  config.vm.provision "shell", path: "scripts/provision/finalize.sh"
end
