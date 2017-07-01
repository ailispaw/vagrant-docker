
if Vagrant.has_plugin?("vagrant-triggers") then
  Vagrant.require_version ">= 1.7.0"
else
  Vagrant.require_version ">= 2.1.0"
end

$vm_gui = false
$vm_memory = 2048
$vm_cpus = 8


$docker_version = "18.06.0-ce"
$vm_ip_address = "172.17.8.101"
$docker_net = "172.16.0.0/12"

def vm_gui
  $vb_gui.nil? ? $vm_gui : $vb_gui
end

def vm_memory
  $vb_memory.nil? ? $vm_memory : $vb_memory
end

def vm_cpus
  $vb_cpus.nil? ? $vm_cpus : $vb_cpus
end

# A dummy plugin for Barge to set hostname and network correctly at the very first `vagrant up`
module VagrantPlugins
  module GuestLinux
    class Plugin < Vagrant.plugin("2")
      guest_capability("linux", "change_host_name") { Cap::ChangeHostName }
      guest_capability("linux", "configure_networks") { Cap::ConfigureNetworks }
    end
  end
end

Vagrant.configure("2") do |config|
  config.vm.define "vagrant-docker"
  config.vm.box = "ailispaw/barge"
  config.vm.hostname = "vagrant-docker"
  config.vm.network :private_network, ip: "#{$vm_ip_address}"
  config.vm.synced_folder ".", Dir.pwd, id: "home", type: "nfs",
    mount_options: [
      'noatime,soft,nolock,vers=3,udp,proto=udp,udp',
      'rsize=8192,wsize=8192,namlen=255,timeo=10,retrans=3,nfsvers=3,actimeo=1'
    ]

  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
    vb.gui = vm_gui
    vb.memory = vm_memory
    vb.cpus = vm_cpus
  end

  config.vm.network :forwarded_port, guest: 2375, host: 2375, auto_correct: true
  config.vm.network :forwarded_port, guest: 9000, host: 9000, auto_correct: true

  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  # Adjusting datetime before provisioning.
  config.vm.provision "timesync", type: "shell", run: "always" do |sh|
    sh.inline = "sntp -4sSc pool.ntp.org; date"
  end

  config.vm.provision :file, source: "./docker", destination: '/tmp/docker'

  config.vm.provision :shell do |sh|
    sh.inline = <<-EOT
      mv /tmp/docker /etc/default/docker
      /etc/init.d/docker restart #{$docker_version}
    EOT
  end

  config.vm.provision :docker do |d|
    d.pull_images "ailispaw/dnsdock"
    d.run "dnsdock",
      image: "ailispaw/dnsdock",
      args: "-v /var/run/docker.sock:/var/run/docker.sock -p 0.0.0.0:53:53/udp"
  end

  # http://portainer.io
  config.vm.provision :docker do |d|
    d.pull_images "portainer/portainer"
    d.run "portainer",
      image: "portainer/portainer",
      args: "-v /var/run/docker.sock:/var/run/docker.sock -p 9000:9000"
  end

  config.vm.provision :shell do |sh|
    sh.inline = <<-EOT
      echo "nameserver 127.0.0.1" > /etc/resolv.conf.head
      dhcpcd -x eth0 && dhcpcd eth0
    EOT
  end

  if Vagrant.has_plugin?("vagrant-triggers") then
    config.trigger.after [:up, :resume] do
      info "Setup DNS resolver and routing to #{$docker_net} domain."
      run <<-EOT
        sh -c "sudo mkdir -p /etc/resolver && \
          echo nameserver #{$vm_ip_address} | sudo tee /etc/resolver/docker && \
          sudo route -n add -net #{$docker_net} #{$vm_ip_address}"
      EOT
    end

    config.trigger.after [:destroy, :suspend, :halt] do
      info "Remove DNS resolver and routing to #{$docker_net} domain."
      run <<-EOT
        sh -c "sudo rm -f /etc/resolver/docker && \
          sudo route -n delete -net #{$docker_net} #{$vm_ip_address}"
      EOT
    end
  else
    config.trigger.after [:up, :resume] do |trigger|
      trigger.info = "Setup DNS resolver and routing to #{$docker_net} domain."
      trigger.run = {
        inline: <<-EOT
          sh -c "sudo mkdir -p /etc/resolver && \
            echo nameserver #{$vm_ip_address} | sudo tee /etc/resolver/docker && \
            sudo route -n add -net #{$docker_net} #{$vm_ip_address}"
        EOT
      }
    end

    config.trigger.after [:destroy, :suspend, :halt] do |trigger|
      trigger.info = "Remove DNS resolver and routing to #{$docker_net} domain."
      trigger.run = {
        inline: <<-EOT
          sh -c "sudo rm -f /etc/resolver/docker && \
            sudo route -n delete -net #{$docker_net} #{$vm_ip_address}"
        EOT
      }
    end
  end
end
