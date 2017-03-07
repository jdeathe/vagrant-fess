# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'digest/sha1'
require 'fileutils'

Vagrant.require_version ">= 1.6.0"

$cloud_config_path = File.expand_path(
  "./user-data",
  File.dirname(__FILE__)
)
$config_path = File.expand_path(
  "./config.rb",
  File.dirname(__FILE__)
)

# Create a hash of the user-data file so changes result in a new instance-id
$cloud_config_hash = Digest::SHA1.file(
  $cloud_config_path
).hexdigest
$cloudinit_uid = $cloud_config_hash[0,12]

# Defaults for config options
$image_version = "7.3.0"
$share_home = true
$vm_name = "fess"
$vm_gui = false
$vm_memory = 1024
$vm_cpus = 1
$shared_folders = {}
$forwarded_ports = {}
$ip_address = "192.168.80.100"

if File.exist?($config_path)
  require $config_path
end

Vagrant.configure(2) do |config|
  config.vm.box = "jdeathe/centos-7-x86_64-minimal-en_us"
  config.vm.box_version = "= %s" % $image_version

  config.vm.define $vm_name do |config|
    config.vm.hostname = $vm_name
    config.vm.post_up_message = "Access FESS on: 'http://%s'. Note: Allow some time for initialisation." % $ip_address

    # Configure SSH port forwarding to start at port 2250
    config.vm.network "forwarded_port", 
      guest: 22, 
      host: 2250, 
      id: "ssh", 
      auto_correct: true

    $forwarded_ports.each do |guest, host|
      config.vm.network "forwarded_port", 
        guest: guest, 
        host: host, 
        auto_correct: true
    end

    # Create a private network, which allows host-only access to the machine
    # using a specific IP.
    config.vm.network "private_network", 
      ip: $ip_address

    # Create a public network, which generally matched to bridged network.
    # Bridged networks make the machine appear as another physical device on
    # your network.
    # config.vm.network "public_network"

    $shared_folders.each_with_index do |(host_folder, guest_folder), index|
      config.vm.synced_folder host_folder.to_s, guest_folder.to_s, 
        id: "shared-folder-%02d" % index, 
        :nfs => true, 
        :mount_options => ["nolock","vers=3","udp"]
    end

    if $share_home
      config.vm.synced_folder ENV['HOME'], ENV['HOME'], 
        id: "home", 
        :nfs => true, 
        :mount_options => ["nolock","vers=3","udp"]
    end

    config.vm.provider "virtualbox" do |vb|
      vb.gui = $vm_gui
      vb.memory = $vm_memory
      vb.cpus = $vm_cpus
      vb.name = $vm_name

      # Remove checks for features not included/required in the base box
      vb.check_guest_additions = false
      vb.functional_vboxsf = false
    end

    if File.exist?($cloud_config_path)
      config.vm.provision "file", 
        source: "#{$cloud_config_path}", 
        destination: "/tmp/user-data"
      config.vm.provision "shell", keep_color: true, 
        name: "Install cloud-init", 
        inline: "yum -y install cloud-init"
      config.vm.provision "shell", keep_color: true, 
        name: "Disabling cloud-init locale module", 
        inline: "sed -i -e 's~^\\([ ]*\\)\\(- locale\\)$~\\1#\\2~' /etc/cloud/cloud.cfg"
      config.vm.provision "shell", keep_color: true, 
        name: "Make cloud-init datastore directory", 
        inline: "mkdir -p /var/lib/cloud/seed/nocloud-net"
      config.vm.provision "shell", keep_color: true, 
        name: "Populate cloud-init meta-data", 
        inline: "echo -e 'hostname: #{$vm_name}\ninstance-id: #{"iid-%s" % $cloudinit_uid}' > /var/lib/cloud/seed/nocloud-net/meta-data"
      config.vm.provision "shell", keep_color: true, 
        name: "Populate cloud-init user-data", 
        inline: "mv /tmp/user-data /var/lib/cloud/seed/nocloud-net/user-data"
      config.vm.provision "shell", keep_color: true, 
        name: "Apply the settings specified in cloud-config", 
        inline: "systemctl restart cloud-config.service || true"
      config.vm.provision "shell", keep_color: true, 
        name: "Execute cloud user/final scripts", 
        inline: "systemctl restart --no-block cloud-final.service"
      config.vm.provision "shell", keep_color: true, 
        name: "Replacing centos user's public key", 
        inline: "cat /home/vagrant/.ssh/authorized_keys > /home/centos/.ssh/authorized_keys"
      config.vm.provision "shell", keep_color: true, 
        name: "Waiting for fess activation", 
        inline: "sleep 1; while ! systemctl -q is-active fess; do sleep 1; done"

      # Use the cloud-config provisioned user with vagrant ssh
      if %w(ssh ssh-config).include? ARGV[0]
        config.ssh.username = "centos"
        private_key_paths = Array.new
        if File.exist?(File.expand_path(
            "./.vagrant/machines/%s/virtualbox/private_key" % $vm_name
        ))
          private_key_paths.push(File.expand_path(
            "./.vagrant/machines/%s/virtualbox/private_key" % $vm_name
          ))
        end
        if File.exist?(File.expand_path(
          "%s/.vagrant.d/insecure_private_key" % ENV['HOME']
        ))
          private_key_paths.push(File.expand_path(
            "%s/.vagrant.d/insecure_private_key" % ENV['HOME']
          ))
        end
        config.ssh.private_key_path = private_key_paths
      end
    end
  end
end
