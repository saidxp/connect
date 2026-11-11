# Gateway VM connecting two LANs (subnet A and subnet B) to the WAN (NAT) <!()!>

Vagrant.configure("2") do |config|

  config.vm.box = "debian13-local"
  # ---------------------------------------------------------
  # Gateway: 3 cards
  #   NIC 1 -> WAN (NAT)          -> enp0s3
  #   NIC 2 -> subnet A (cable)   -> enp0s8
  #   NIC 3 -> subnet B (cable)   -> enp0s9
  # ---------------------------------------------------------
  
  config.vm.define "gateway" do |gateway|
    gateway.vm.provider "virtualbox" do |vb|
      vb.name = "connect-gateway"

      # NIC 1 -> WAN
      vb.customize ["modifyvm", :id, "--nic1", "nat"]

      # NIC 2 -> subnet A
      vb.customize ["modifyvm", :id, "--nic2", "intnet"]
      vb.customize ["modifyvm", :id, "--intnet2", "gateway-lan"]

      # NIC 3 -> subnet B
      vb.customize ["modifyvm", :id, "--nic3", "intnet"]
      vb.customize ["modifyvm", :id, "--intnet3", "gateway-lan-b"]

    end
    gateway.vm.provision "shell", path: "provision/provision.sh"
    gateway.vm.provision "shell", path: "provision/gateway.sh"
    gateway.vm.provision "shell", path: "provision/dhcp.sh"
    gateway.vm.provision "shell", path: "provision/dns.sh" 
  end

  # ---------------------------------------------------------
  # Client 1: subnet A (gets its IP by DHCP)
  # ---------------------------------------------------------
  config.vm.define "client1" do |client|
    client.vm.provider "virtualbox" do |vb|
      vb.name = "connect-client1"

      # NIC 1 -> NAT (Vagrant SSH)
      vb.customize ["modifyvm", :id, "--nic1", "nat"]

      # NIC 2 -> same cable as the gateway's subnet A card
      vb.customize ["modifyvm", :id, "--nic2", "intnet"]
      vb.customize ["modifyvm", :id, "--intnet2", "gateway-lan"]
      vb.customize ["modifyvm", :id, "--macaddress2", "080027AA0001"]

    end
    client.vm.provision "shell", path: "provision/provision.sh"
    client.vm.provision "shell", path: "provision/client.sh"
    client.vm.provision "shell", inline: "echo rebooting to apply dhcpcd settings", reboot: true
  end

  # ---------------------------------------------------------
  # Client 2: subnet B (static IP)
  # ---------------------------------------------------------
  config.vm.define "client2" do |client|
    client.vm.provider "virtualbox" do |vb|
      vb.name = "connect-client2"

      # NIC 1 -> NAT (Vagrant SSH)
      vb.customize ["modifyvm", :id, "--nic1", "nat"]

      # NIC 2 -> same cable as the gateway's subnet B card
      vb.customize ["modifyvm", :id, "--nic2", "intnet"]
      vb.customize ["modifyvm", :id, "--intnet2", "gateway-lan-b"]
    end

    client.vm.provision "shell", path: "provision/provision.sh"
    client.vm.provision "shell", path: "provision/client-b.sh"
  end

end