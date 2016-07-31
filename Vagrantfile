require 'vagrant-service-manager'
require 'vagrant-registration'

# -*- mode: ruby -*-
# vi: set ft=ruby :

# Usage:
#
# To use this Vagrantfile:
# * Have valid RH employee subscription account
# * Be connected to the internal Red Hat network
# * Have the 'vagrant-registration' plugin installed
# * Have the 'vagrant-service-manager' plugin installed

# General configuration:

# URLs from where to fetch the Vagrant VirtualBox images
# FIXME: How to point to the official box image?
VAGRANT_VIRTUALBOX_URL='http://cdk-builds.usersys.redhat.com/builds/weekly/13-Apr-2016.rc3/rhel-cdk-kubernetes-7.2-23.x86_64.vagrant-virtualbox.box'
VAGRANT_LIBVIRT_URL='http://cdk-builds.usersys.redhat.com/builds/weekly/13-Apr-2016.rc3/rhel-cdk-kubernetes-7.2-23.x86_64.vagrant-libvirt.box'

# The IP address and hostname of the VM as exposed on the host.
# Uses xip.io for now to have zero configuration routes
PUBLIC_ADDRESS="10.1.2.2"
REGISTRY_HOST="hub.openshift.#{PUBLIC_ADDRESS}.xip.io"

# The amount of memory available to the VM
VM_MEMORY = ENV['VM_MEMORY'] || 3072

# Number of VM CPUs
VM_CPU = ENV['VM_CPU'] || 2

# The Docker registry from where we pull the OpenShift Docker image
DOCKER_REGISTRY="brew-pulp-docker01.web.prod.ext.phx2.redhat.com:8888"

# The name of the OpenShift image.
IMAGE_NAME="openshift3/ose"
IMAGE_TAG="v3.2.0.8"

SUBSCRIPTION_INFO = "
    Red Hat subscription credentials are needed for this VM.
    You can supply them interactively or by setting environment variables.
    Set these environment variables to your subscription username/password to avoid interactive registration:

    $ export SUB_USERNAME=rhn-username
    $ export SUB_PASSWORD=password
  "

REQUIRED_PLUGINS = %w(vagrant-service-manager vagrant-registration vagrant-sshfs)
message = ->(name) { "#{name} plugin is not installed, run `vagrant plugin install #{name}` to install it." }
errors = []
# Validate and collect error message if plugin is not installed
REQUIRED_PLUGINS.each {|plugin| errors << message.call(plugin) unless Vagrant.has_plugin?(plugin) }
msg = errors.size > 1 ? "Errors: \n* #{errors.join("\n* ")}" : "Error: #{errors.first}"
raise Vagrant::Errors::VagrantError.new, msg unless errors.size == 0

Vagrant.configure(2) do |config|

  config.vm.define "cdk" do |cdk|
      cdk.vm.provider "virtualbox" do |v, override|
      	override.vm.box = "cdk_v2"
        override.vm.box_url = "#{VAGRANT_VIRTUALBOX_URL}"
        #v.name = "openshift.cdk-2"
        v.memory = VM_MEMORY
        v.cpus   = VM_CPU
        v.customize ["modifyvm", :id, "--ioapic", "on"]
        v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      end

      cdk.vm.provider "libvirt" do |v, override|
      	override.vm.box = "cdk_v2"
        override.vm.box_url = "#{VAGRANT_LIBVIRT_URL}"
        v.driver = "kvm"
        v.memory = VM_MEMORY
        v.cpus   = VM_CPU
      end

      cdk.vm.network "private_network", ip: "#{PUBLIC_ADDRESS}"

      if ENV.has_key?('SUB_USERNAME') && ENV.has_key?('SUB_PASSWORD')
        cdk.registration.username = ENV['SUB_USERNAME']
        cdk.registration.password = ENV['SUB_PASSWORD']
        # After provisioning registration can be skipped as well as un-registering on halt
        #cdk.registration.skip = true
        #cdk.registration.unregister_on_halt = false
      end

      case ARGV[0]
      when "up", "halt"
        if cdk.registration.username.nil? || cdk.registration.password.nil?
          puts SUBSCRIPTION_INFO
        end
      end

      cdk.vm.synced_folder '.', '/vagrant', disabled: true
      if Vagrant::Util::Platform.windows?
        target_path = ENV['USERPROFILE'].gsub(/\\/,'/').gsub(/[[:alpha:]]{1}:/){|s|'/' + s.downcase.sub(':', '')}
        cdk.vm.synced_folder ENV['USERPROFILE'], target_path, type: 'sshfs', sshfs_opts_append: '-o umask=000 -o uid=1000 -o gid=1000'
      else
        cdk.vm.synced_folder ENV['HOME'], ENV['HOME'], type: 'sshfs', sshfs_opts_append: '-o umask=000 -o uid=1000 -o gid=1000'
      end
      cdk.vm.provision "shell", inline: <<-SHELL
        sudo setsebool -P virt_sandbox_use_fusefs 1
      SHELL

      # Set OPENSHIFT_VAGRANT_USE_LANDRUSH=true in your shell in order to use Landrush
      if Vagrant.has_plugin?('landrush')
        cdk.landrush.enabled = true
        tld = 'openshift.cdk'
        cdk.landrush.tld = tld
        cdk.landrush.host tld, "#{PUBLIC_ADDRESS}"
        cdk.vm.provision :shell, inline: <<-SHELL
          sed -i.orig -e "s/OPENSHIFT_SUBDOMAIN=.*/OPENSHIFT_SUBDOMAIN='#{tld}'/g" /etc/sysconfig/openshift_option
        SHELL
      end

      # Set OPENSHIFT_VAGRANT_USE_OSE_3_2=true in your shell to use OSE 3.2 instead of 3.1
      if ENV.fetch('OPENSHIFT_VAGRANT_USE_OSE_3_2', 'false').to_s.downcase == 'true'
        cdk.vm.provision :shell, inline: <<-SHELL
          sed -i.orig -e "s/ADD_REGISTRY=.*/ADD_REGISTRY='--add-registry registry.access.redhat.com --add-registry brew-pulp-docker01.web.prod.ext.phx2.redhat.com:8888'/g" /etc/sysconfig/docker
          add_insecure_registry -r brew-pulp-docker01.web.prod.ext.phx2.redhat.com:8888
          DOCKER_REGISTRY=#{DOCKER_REGISTRY} IMAGE_TAG=#{IMAGE_TAG} IMAGE_NAME=#{IMAGE_NAME} /usr/bin/sccli openshift
        SHELL
      end

      # Work around for avoiding that vagrant-service-manager does automatic start of OpenShift via sccli
      # https://github.com/projectatomic/adb-atomic-developer-bundle/issues/342
      cdk.servicemanager.services = "docker"
      cdk.vm.provision "shell", inline: <<-SHELL
        systemctl enable openshift 2>&1
        systemctl start openshift
      SHELL

      cdk.vm.provision "shell", inline: <<-SHELL
        echo
        echo "Successfully started and provisioned VM with #{VM_CPU} cores and #{VM_MEMORY} MB of memory."
        echo "To modify the number of cores and/or available memory set the environment variables"
        echo "VM_CPU respectively VM_MEMORY."
        echo
        echo "You can now access the OpenShift console on: https://#{PUBLIC_ADDRESS}:8443/console"
        echo
        echo "To use OpenShift CLI, run:"
        echo "$ vagrant ssh"
        echo "$ oc login #{PUBLIC_ADDRESS}:8443"
        echo
        echo "Configured users are (<username>/<password>):"
        echo "openshift-dev/devel"
        echo "admin/admin"
        echo
        echo "If you have the oc client library on your host, you can also login from your host."
        echo
      SHELL
    end
end
