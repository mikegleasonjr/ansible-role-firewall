boxes = {
  "ubuntu/trusty64" => {
    :ip  => '192.168.33.10',
    :cpu => "2",
    :ram => "256"
  },
  "ubuntu/xenial64" => {
    :ip  => '192.168.33.11',
    :cpu => "2",
    :ram => "256"
  },
  "centos/7" => {
    :ip  => '192.168.33.12',
    :cpu => "2",
    :ram => "256"
  },
}

Vagrant.configure("2") do |config|
  boxes.each do |box, options|
    config.vm.define box.dup.sub!("/", "-") do |machine|
      machine.vm.box = box
      machine.vm.box_check_update = false
      machine.vm.network :private_network, ip: options[:ip]

      machine.vm.provider "virtualbox" do |vb|
        vb.memory = options[:ram]
        vb.cpus = options[:cpu]
      end

      machine.vm.provision "ansible" do |ansible|
        ansible.playbook = "tests.yml"
      end
    end
  end
end
