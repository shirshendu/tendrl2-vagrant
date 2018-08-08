# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'

# Some variables we need below
VAGRANT_ROOT = File.dirname(File.expand_path(__FILE__))

# Disable parallel runs - breaks peer probe in the end
ENV['VAGRANT_NO_PARALLEL'] = 'yes'

#################
# Poor man's OS detection routine
#################
module OS
  def self.windows?
    (/cygwin|mswin|mingw|bccwin|wince|emx/ =~ RUBY_PLATFORM) != nil
  end

  def self.mac?
    (/darwin/ =~ RUBY_PLATFORM) != nil
  end

  def self.linux?
    !OS.windows? && !OS.mac?
  end
end

#################
# Set vagrant default provider according to OS detected
#################
if OS.windows?
  ENV['VAGRANT_DEFAULT_PROVIDER'] = 'virtualbox'
elsif OS.mac?
  ENV['VAGRANT_DEFAULT_PROVIDER'] = 'virtualbox'
elsif OS.linux?
  ENV['VAGRANT_DEFAULT_PROVIDER'] = 'libvirt' # '<your_provider_name>'
end

#################
# General VM settings applied to all VMs
#################
VMCPU = 2         # number of cores per VM
VMMEM = 512      # amount of memory in MB per VM
VMDISK = '256m'.freeze # size of brick disks in MB
# Metadata volume isn't generated if <200MB:
# https://github.com/gluster/gdeploy/blob/0462ad54f1d8f9c83502e774246a528ae2c8c83f/modules/lv.py#L168

#################

storage_node_count = -1
disk_count = -1

tendrl_conf = YAML.load_file 'tendrl2.conf.yml'
storage_node_count = tendrl_conf['storage_node_count'].to_i
disk_count = tendrl_conf['disk_count'].to_i

if storage_node_count < 2
  puts 'Minimum 2 nodes needed'
  exit 1
end

if disk_count < 1
  puts 'Minimum 1 disk needed'
  exit 1
end

if ARGV[0] != 'ssh-config' && ARGV[0] != 'ssh'
  puts 'Detected settings from tendrl2.conf.yml:'
  puts "  We have configured #{storage_node_count} VMs, each with #{disk_count} disks"
  puts "  Storage pool: #{ENV['LIBVIRT_STORAGE_POOL'] || 'default'}" if ENV['VAGRANT_DEFAULT_PROVIDER'] == 'libvirt'
end

def vb_attach_disks(disks, provider, boxName)
  (1..disks).each do |i|
    file_to_disk = File.join VAGRANT_ROOT, 'disks', "#{boxName}-disk#{i}.vdi"
    unless File.exist?(file_to_disk)
      provider.customize [
        'createhd',
        '--filename', file_to_disk,
        '--size', VMDISK.to_i
      ]
    end
    provider.customize [
      'storageattach',
      :id,
      '--storagectl', 'SATA Controller',
      '--port', i,
      '--device', 0,
      '--type', 'hdd',
      '--medium', file_to_disk
    ]
  end
end

def libvirt_attach_disks(disks, provider)
  (1..disks).each do
    provider.storage :file, bus: 'virtio', size: VMDISK
  end
end

# Vagrant config section starts here
Vagrant.configure(2) do |config|
  config.vm.box = 'centos/7'

  config.vm.provider 'virtualbox' do |vb, _override|
    vb.gui = false
  end

  config.vm.provider 'libvirt' do |libvirt, _override|
    # Use virtio device drivers
    libvirt.nic_model_type = 'virtio'
    libvirt.disk_bus = 'virtio'
    libvirt.storage_pool_name = ENV['LIBVIRT_STORAGE_POOL'] || 'default'
  end

  config.vm.network 'private_network', type: 'dhcp', auto_config: true

  (1..storage_node_count).each do |node_index|
    config.vm.define "gd2-#{node_index}" do |machine|
      # Provider-independent options
      machine.vm.hostname = "gd2-#{node_index}"
      machine.vm.synced_folder 'builds', '/vagrant', type: "rsync"

      machine.vm.provider 'virtualbox' do |vb, override|
        # Make this a linked clone for cow snapshot based root disks
        vb.linked_clone = true

        # Set VM resources
        vb.memory = VMMEM
        vb.cpus = VMCPU

        # Don't display the VirtualBox GUI when booting the machine
        vb.gui = false

        # give this VM a proper name
        vb.name = "gd2-#{node_index}"

        # attach brick disks
        vb_attach_disks(disk_count, vb, "gd2-#{node_index}")

        # Accelerate SSH / Ansible connections (https://github.com/mitchellh/vagrant/issues/1807)
        vb.customize ['modifyvm', :id, '--natdnshostresolver1', 'on']
        vb.customize ['modifyvm', :id, '--natdnsproxy1', 'on']
      end

      machine.vm.provider 'libvirt' do |libvirt, override|
        # Set VM resources
        libvirt.memory = VMMEM
        libvirt.cpus = VMCPU

        # Use virtio device drivers
        libvirt.nic_model_type = 'virtio'
        libvirt.disk_bus = 'virtio'

        # connect to local libvirt daemon as root
        libvirt.username = 'root'

        # attach brick disks
        libvirt_attach_disks(disk_count, libvirt)
      end

      if node_index == storage_node_count
        machine.vm.provision :prepare_env, type: :ansible do |ansible|
          ansible.limit = 'all'
          ansible.groups = {
            'gluster4-servers' => ["gd2-[1:#{storage_node_count}]"],
          }
          ansible.playbook = 'ansible/prepare-environment.yml'
        end

        machine.vm.provision :create_cluster, type: :ansible, run: :never do |ansible|
          ansible.limit = 'all'
          ansible.groups = {
            'gluster4-servers' => ["gd2-[1:#{storage_node_count}]"],
          }
          ansible.playbook = 'ansible/create-cluster.yml'
        end

        machine.vm.provision :refresh_gluster, type: :ansible, run: :never do |ansible|
          ansible.limit = 'all'
          ansible.groups = {
            'gluster4-servers' => ["gd2-[1:#{storage_node_count}]"]
          }
          ansible.playbook = 'ansible/refresh-gluster.yml'
        end
      end
    end
  end
end
