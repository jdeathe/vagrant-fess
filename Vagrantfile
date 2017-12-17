# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'digest/sha1'
require 'fileutils'

Vagrant.require_version ">= 1.9.2"

$config_path = File.expand_path(
  "./config.rb",
  File.dirname(__FILE__)
)
$cloud_config_meta_path = File.expand_path(
  "./cidata/meta-data",
  File.dirname(__FILE__)
)
$cloud_config_user_path = File.expand_path(
  "./cidata/user-data",
  File.dirname(__FILE__)
)

# Create a hash of the user-data file so changes result in a new instance-id
$cloud_config_user_hash = Digest::SHA1.file(
  $cloud_config_user_path
).hexdigest
$cloudinit_uid = $cloud_config_user_hash[0,12]

# Defaults for config options
$image_version = "7.4.1"
$linked_clone = false
$share_home = false
$vm_name = "fess.local"
$vm_gui = false
$vm_memory = 1024
$vm_cpus = 1
$shared_folders = {}
$forwarded_ports = {}
$ip_address = "192.168.80.100"

if File.exist?($config_path)
  require $config_path
end

if %w(up reload).include? ARGV[0]
  File.open(
    "#{$cloud_config_meta_path}",
    "w"
  ) do |meta|
    meta.write "# Auto-Generated - DO NOT CHANGE THIS\n"
    meta.write "hostname: #{$vm_name}\n"
    meta.write "instance-id: iid-#{$cloudinit_uid}"
  end

  # Generate the NoCloud ISO
  system("
    if command -v hdiutil &> /dev/null; then
      hdiutil makehybrid \
        -quiet \
        -ov \
        -hfs \
        -iso \
        -joliet \
        -default-volume-name cidata \
        -o iso/nocloud.iso \
        cidata/
    elif command -v mkisofs &> /dev/null; then
      mkisofs \
        -R \
        -J \
        -V cidata \
        -o iso/nocloud.iso \
        cidata/*
    fi
  ")
end

$cloud_config_iso = File.expand_path(
  "./iso/nocloud.iso",
  File.dirname(__FILE__)
)

if !File.exist?($cloud_config_iso)
  puts "ERROR: Missing NoCloud ISO file: 
%s" % $cloud_config_iso
  abort
end

$hostupdater_message = ""
if !Vagrant::Util::Platform.windows?
  unless Vagrant.has_plugin?("vagrant-hostsupdater")
    $hostupdater_message =" either run 'vagrant plugin install vagrant-hostsupdater && vagrant reload',
"
  end
end

Vagrant.configure(2) do |config|
  config.vm.box = "jdeathe/centos-7-x86_64-minimal-cloud-init-en_us"
  config.vm.box_version = "= #{$image_version}"

  config.vm.define $vm_name do |config|
    config.vm.hostname = $vm_name
    config.vm.post_up_message = "After the machine booted Cloud-Init applies the changes 
defined in user-data.

For access%s add a /etc/hosts or DNS entry: '%s %s'.

The service can take a few minutes to start. Run:
  vagrant ssh -- sudo tail -f /var/log/fess/server_0.log

Proceed once you see a 'Boot successful' info message.
If it hangs you can try restarting Fess:
  vagrant ssh -- sudo systemctl restart fess

To complete the installation, visit: 
  http://%s" % [
      $hostupdater_message,
      $ip_address,
      $vm_name,
      $vm_name
    ]

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
      vb.cpus = $vm_cpus
      vb.gui = $vm_gui
      vb.linked_clone = $linked_clone
      vb.memory = $vm_memory
      vb.name = $vm_name

      # Remove checks for features not included/required in the base box
      vb.check_guest_additions = false
      vb.functional_vboxsf = false

      if File.exist?($cloud_config_iso)
        vb.customize [
          "storageattach", :id,
          "--storagectl", "SATA Controller",
          "--port", "1",
          "--type", "dvddrive",
          "--medium", "#{$cloud_config_iso}"
        ]
      end
    end

    if File.exist?($cloud_config_iso)
      config.vm.provision "shell", keep_color: true,
        name: "Wait for Cloud-init boot-finished",
        inline: "tail -f /var/log/cloud-init-output.log & \
          CIO_PID=${!}; \
          until [[ -e /var/lib/cloud/instance/boot-finished ]]; do \
            sleep 1; \
          done; \
          kill ${CIO_PID}"
      config.vm.provision "shell", keep_color: true,
        name: "Cloud-init success verification",
        inline: "if ! grep -qE \
            '\\\"errors\\\": \\[\\]' \
            /var/lib/cloud/data/result.json; \
          then \
            printf -- 'ERROR Cloud-init failed\\n%s' \
            \"$(
              cat /var/lib/cloud/data/result.json
            )\" 1>&2; \
            exit 1; \
          fi"
    end
  end
end

if %w(up reload).include? ARGV[0]
  File.open(
    "#{$cloud_config_meta_path}",
    "w"
  ) do |meta|
    meta.write "# Auto-Generated - DO NOT CHANGE THIS"
  end
end
