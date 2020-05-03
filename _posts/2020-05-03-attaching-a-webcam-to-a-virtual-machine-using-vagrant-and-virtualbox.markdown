---
layout: post
title:  "Attaching a webcam to a virtual machine using Vagrant and VirtualBox"
categories: linux windows virtualization
---
It all started with the virtual chroma screen function of Zoom,
which does not work on older laptops that do not have a beefy CPU. So, one day on Hacker News,
I found [an article by Ben Elder](https://elder.dev/posts/open-source-virtual-background/) that describes how to
set up your own virtual webcam on Linux.

It is rather easy to set up a virtual Linux machine on my laptop.
I like to use [Vagrant](https://www.vagrantup.com/) to script my virtual machines. Vagrant can work
with several backends to provision VMs; I prefer [Oracle VirtualBox](https://www.virtualbox.org/).

With Vagrant, all definitions for bringing up a VM are contained within the Vagrantfile, which is basically
a Ruby script. I was wondering if there is a way to connect the host computer's webcam to the guest VM
on startup. Turns out there is, but it is not quite trivial.

From the host command line, you would call

  ```
  VBoxManage controlvm {machine-uuid} webcam attach {webcam-id}
  ```

That's what I am going to automate! My first idea was to set up the webcam during the provisioning
phase, where Vagrant has an easy way to call the VBoxManage interface. Something like this:

  ``` ruby
  config.vm.provider "virtualbox" do |vb|
    ...
    vb.customize [ "controlvm", "zoombox", "webcam", "attach", ".0" ]
  end
  ```

Turns out this does not work. At the point in time this is called, the VM is not up yet and you cannot 
use `controlvm`.

There is [a GitHub repository by Kenta Yonekura](https://github.com/yoneken/Win-vagrant-Ubuntu-webcam)
that pointed me to the right solution. Vagrant has the possibility to use _triggers_ that can be called
before or after certain commands. However Kenta's solution did not work for me, for these reasons:

  - The ID of the virtual machine should be read from an ID file in the `.vagrant` directory. This file
    does not exist on my system when the trigger is called.
  - The `VBoxManage.exe` executable is not in my `PATH`, so it cannot be found.

I managed to work around the first issue by assigning an explicit name to my VM. This name is 
currently hardwired in my `Vagrantfile`. Also, I noticed that VirtualBox defines an environment
variable `VBOX_MSI_INSTALL_PATH`, which points to the VirtualBox binaries.

Finally, I had to handle some issues around shell quoting with Windows - this one was easily fixed by using
the `run` and `args` options instead of `inline`.

So here is my code:

  ``` ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|

  # Use Debian Stretch - I had issues with package hashes when using Python 3 on Debian Buster
  config.vm.box = "debian/contrib-stretch64"

  config.vm.hostname = "zoombox.localdomain"
  config.vm.network "private_network", ip: "192.168.17.31"
  config.ssh.forward_x11 = true

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"
    # "--natdnshostresolver1": Fix DNS for use with VPN tunnel,
    # see http://askubuntu.com/questions/238040/how-do-i-fix-name-service-for-vagrant-client.
    # Most of the other options are courtesy of 
    # https://github.com/yoneken/Win-vagrant-Ubuntu-webcam.
    vb.customize [ "modifyvm", :id,
      "--name", "zoombox",
      "--vram", "256",
      "--accelerate3d", "on",
      "--clipboard", "bidirectional",
      "--hwvirtex", "on",
      "--nestedpaging", "on",
      "--largepages", "on",
      "--ioapic", "on",
      "--pae", "on",
      "--paravirtprovider", "kvm",
      "--natdnshostresolver1", "on",
      "--usb", "on",
      "--usbehci", "on",      
    ]
  end

  config.vm.provision :shell do |s|
    s.path = File.join( Dir.pwd, "provisioner.sh" )
  end
  
  config.trigger.after [ :up, :reload ] do |t|
    # Attach webcam
    # Possible upgrade: Select webcam instead of just using the default ".0",
    # like using [VBoxManage list webcams]
    # Note: VirtualBox does not put itself into PATH and I don't wnat to change that,
    # so read out the install path from the corresponding environment variable
    t.info = "Mount webcam to \"zoombox\". VirtualBox path is #{ENV['VBOX_MSI_INSTALL_PATH']}"
    t.run = {
      path: File.join( ENV['VBOX_MSI_INSTALL_PATH'], "VBoxManage.exe" ),
      args: [ "controlvm", "zoombox", "webcam", "attach", ".0" ],
    }
  end

end
  ```

This can also be found in [my GitHub repo](https://github.com/hellmarbecker/vagrant-virtual-background/).

