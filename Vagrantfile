# -*- mode: ruby -*-
# vi: set ft=ruby :

# This is intended to be a drop-in Vagrantfile, which reads VM configurations
# from a yaml file (vagrant.yml) in the root directory.
# It supports vagrant cloud boxes and traditional boxes
# See the README for more thorough documentation.

# We're going to read from yaml files, so we gots to know how to yaml
require 'yaml'

# If the external_functions directory exists, then load
# functions from any ruby files in that directory
#
# NOTE: Sometimes we want to do things that are a bit more dynamic than just
#       setting Vagrantfile properties. This provides the ability to define
#       this sort of dynamic behavior externally to the Vagrantfile itself
root_dir = File.dirname(__FILE__)
external_functions_dir = 'external_functions'
external_functions_path = "#{root_dir}/#{external_functions_dir}"

if File.exist?( external_functions_path )
  external_function_files = Dir.glob( "#{external_functions_path}/*.rb") 
  external_function_files && external_function_files.each do |external_function_file|
    require_relative "#{external_functions_dir}/#{File.basename( external_function_file, '.rb' )}"
  end
end

###############################################################################
# Utility functions
###############################################################################

# Print an error message and stop execution on handled errors
def handle_error(error_msg)
  puts "ERROR: #{error_msg}"
  exit
end

# Check the "nodes" element from vagrant.yml for existence and completeness
def verify_nodes(nodes)
  # Make sure that at least one node is defined
  if !nodes || nodes.empty?
    error_msg = 'No nodes defined in vagrant.yml'
    handle_error(error_msg)
  end

  # TODO: Add per-node checks for completeness
  #       Build up one big error message with all failed checks
end

# Convert the shell provisioner arguments from vagrant.yml
# into an array for the vagrant shell provisioner
def shell_provisioner_args(yaml_arguments)
  shell_arguments = Array.new

  # Arguments may or may not be named,
  # and named arguments may or may not have a value.
  yaml_arguments.each do |argument|
    argument.key?('name') && shell_arguments.push(argument['name'])
    argument.key?('value') && shell_arguments.push(argument['value'])
  end

  shell_arguments
end

# convert all keys in the given hash to symbols
# NOTE: Doesn't work with nested hashes, but I don't need this for those yet
def keys_to_symbols(hash_in)
  hash_out = hash_in.inject({}) do |hash_rekeyed, (key, value)|
    hash_rekeyed[key.to_sym] = value
    hash_rekeyed
  end

  hash_out
end

###############################################################################
# VM Configuration functions
# 
# The functions below read node settings from vagrant.yml and set the
# properties of a vagrant VM appropriately.
# They are called from the main vagrant configuration loop. 
###############################################################################

# Configure basic information (box, hostname) for the given node
# from settings in vagrant.yml
def configure_basic_info(node, node_details, boxes)
  # set the box name and url (if not a vagrant cloud box)
  box_name = "#{node_details['box']}"
  node.vm.box = "#{box_name}"
  boxes && boxes.key?("#{box_name}") && node.vm.box_url = boxes[box_name]
     
  # configure basic settings
  node.vm.hostname = node_details['hostname']
end

# Configure networks for the given node from settings in vagrant.yml
def configure_networks(node, node_details)
  networks = node_details['networks']
  networks && networks.each do |network|
    network.each do |network_type, network_params|
      if network_params
        network_params = keys_to_symbols(network_params)
        node.vm.network network_type, network_params
      else
        node.vm.network network_type
      end
    end
  end
end

# Configure synced folders for the given node from settings in vagrant.yml
def configure_synced_folders(node, node_details)
  synced_folders = node_details['synced_folders']
  synced_folders && synced_folders.each do |synced_folder|
    node.vm.synced_folder synced_folder['host'], synced_folder['guest'], type: synced_folder['type']
  end
end

# Configure forwarded ports for the given node from settings defined in vagrant.yml
def configure_forwarded_ports(node, node_details)
  forwarded_ports = node_details['forwarded_ports']
  forwarded_ports && forwarded_ports.each do |forwarded_port|
    forwarded_port = keys_to_symbols(forwarded_port)
    node.vm.network 'forwarded_port', forwarded_port
  end
end

# Configure provisioner properties for the given node from settings defined in vagrant.yml
#
# Each key in vagrant.yml should correspond to a valid vagrant provisioner.
# Each value should correspond to a valid setting for that provisioner.
#   (Except for 'arguments', which is an array of arguments to the shell provisioner script.)
def configure_provisioners(node, node_details)
  provisioners = node_details['provisioners']
  provisioners && provisioners.each do |provisioner|
    provisioner.each do |provisioner_type, provisioner_params|
      node.vm.provision provisioner_type do |provision|
        provisioner_params.each do |key, value|
          if key == 'arguments'
            provision.args = shell_provisioner_args(value) 
          else
            provision.send("#{key}=", value) 
          end
        end
      end
    end
  end
end

# Configure provider-specific settings for the given node from settings defined in vagrant.yml
#
# Each key in vagrant.yml should correspond to a valid vagrant provider
# Each value should be a hash of valid settings for that provider
# NOTE: memory and cpus are common enough settings that I don't treat them as
#       provider-specific in the .yml files
def configure_providers(node, node_details, node_name)
  # General provider-specific settings
  providers = node_details['providers']
  providers && providers.each do |provider_type, provider_params|
    node.vm.provider provider_type do |node_provider|
       provider_params.each do |key, value| 
         node_provider.send("#{key}=", value)
       end
    end
  end

  # Special case provider-specifc settings
  node.vm.provider 'virtualbox' do |vb|
    vb.customize [ 'modifyvm', :id, '--memory', node_details['memory'] ]
    vb.customize [ 'modifyvm', :id, '--cpus', node_details['cpus'] ]
    vb.name = node_name
  end
  node.vm.provider 'vmware_fusion' do |vmf|
    vmf.vmx['memsize'] = node_details['memory']
    vmf.vmx['numvcpus'] = node_details['cpus']
  end
end

# Call any external functions that are defined in vagrant.yml.
# NOTE: These functions must be defined in ruby files in the
#       external_functions folder
def call_external_functions(node, node_details)
  external_functions = node_details['external_functions']
  external_functions && external_functions.each do |external_function|
    send( external_function, node )
  end
end


###############################################################################
# Initialization
###############################################################################

# Verify that vagrant.yml exists
root_dir = File.dirname(__FILE__)
vagrant_yaml_file = "#{root_dir}/vagrant.yml"
error_msg = "#{vagrant_yaml_file} does not exist"
handle_error(error_msg) unless File.exists?(vagrant_yaml_file)

# Read box and node configs from vagrant.yml
vagrant_yaml = YAML.load_file(vagrant_yaml_file)
error_msg = "#{vagrant_yaml_file} exists, but is empty"
handle_error(error_msg) unless vagrant_yaml

boxes = vagrant_yaml['boxes']
nodes = vagrant_yaml['nodes']

# Verify that node definitions exist and are well-formed
verify_nodes(nodes)

###############################################################################
# Vagrant configuration loop
###############################################################################

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = '2'

# For each node defined in vagrant.yml,
# set the properties of a vagrant VM by calling the functions defined above.
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # Define vagrant VMs for each node defined in vagrant.yml
  nodes.each do |node_name, node_details|
    config.vm.define node_name do |node|
      configure_basic_info(node, node_details, boxes)
      configure_networks(node, node_details)
      configure_synced_folders(node, node_details)
      configure_forwarded_ports(node, node_details)
      configure_provisioners(node, node_details)
      configure_providers(node, node_details, node_name)

      call_external_functions(node, node_details)
    end
  end
end
