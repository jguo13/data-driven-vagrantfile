# -*- mode: ruby -*-
# vi: set ft=ruby :

# This is intended to be a drop-in Vagrantfile, which reads VM configurations
# from a yaml file (vagrant.yml) in the root directory.
# It is only compatible with vagrant 1.5+
# See the README for more thorough documentation.

# We're going to read from yaml files, so we gots to know how to yaml
require 'yaml'

# Read box and node configs from vagrant.yml
root_dir = File.dirname(__FILE__)
config = YAML.load_file("#{root_dir}/vagrant.yml")
boxes = config['boxes']
nodes = config['nodes']

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  # Define vagrant VMs for each node defined in nodes.yml
  nodes.each do |node_name, node_details|
    config.vm.define node_name do |node|
      # set box, configure hostname and IP
      node.vm.box = "#{node_details['box']}"
      node.vm.hostname = node_details['hostname']
      node.vm.network "private_network", ip: node_details['ip']

      # configure synced folders 
      synced_folders = node_details['synced_folders']
      synced_folders && synced_folders.each do |synced_folder|
        node.vm.synced_folder synced_folder['source'], synced_folder['dest']
      end

      # configure forwarded ports
      forwarded_ports = node_details['forwarded_ports']
      forwarded_ports && forwarded_ports.each do |forwarded_port|
        node.vm.network "forwarded_port", guest: forwarded_port['guest'], host: forwarded_port['host']
      end

      # configure memory and cpus
      # TODO: extend to work with VMWare Fusion provider
      node.vm.provider :virtualbox do |vb|
        vb.name = node_name
        vb.customize [ "modifyvm", :id, "--memory", node_details['memory'] ]
        vb.customize [ "modifyvm", :id, "--cpus", node_details['cpus'] ]
      end
    end
  end

end