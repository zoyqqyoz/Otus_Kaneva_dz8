MACHINES = {
  :"otusrpm" => {
              :box_name => "centos/8",
              :box_version => "2011.0",
              :cpus => 2,
              :memory => 2048,
            }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
        config.vm.synced_folder ".", "/vagrant", disabled: true
        config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.box_version = boxconfig[:box_version]
      box.vm.host_name = boxname.to_s
      box.vm.provider "virtualbox" do |v|
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
      end
    end
  end
end

