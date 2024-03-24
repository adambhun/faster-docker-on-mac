# Faster Docker on Mac
# What is this?

This document describes how I managed to speed up Docker on a Macbook with an Intel processor.
I do not use a Macbook, unless I have to, so I am not good at using them. I just did this to help out some of my colleagues.

# Background

Docker Desktop on Mac is very inefficient regarding file operations. As far as I know - which is not much -, this is because it archives the whole context, then sends it to the docker daemon which then mounts the whole 'users' folder of the Macbook with NFS. Or at least that's what people who understand IOS better told me.

So the theory behind my solution is that I circumvent these two problems by having a VM managed by Vagrant act something like Docker Desktop. Macs do not natively support Docker so in essence, you would be running your containers in some sort of VM anyways. If you have an M1 or M2 chip this guide probably won't help you because as far as I know, it is currently impossible to get Docker to work on the newest Macbooks.
The tricky part is, that I set up the VM as the NFS server, so the host - the Macbook - is the client. This eliminates the need for having to copy files the way Docker Desktop does.
It also eliminates the problem that Docker Desktop has with NFS; it uses version 4. The problem with that, is that it uses the TCP protocol - and it will not use anything else -, which is much slower than the UDP protocol, which is supported by NFSv3. I could not find a way to force NFSv3 to work on a Macbook with a fresh OS, so I used the VM - which can be any Linux - as the NFS server.
The folder that is shared between the host and the VM is synced automatically, as long as the VM is running.

- Why NFS of all things? Vagrant supports other mounting options for the synced folder!
- Default method: too slow.
  rsync: some lando commands simply did not work
  I'm sure I have tried everything else, just forgot to wrote them all down.

# Caveats

- <mark style="background: #ea3323">When provisioning the VM, the contents of the mounted folder WILL BE LOST!</mark>
- Tested on 2020 MacBook Pro 2GHz Quad-Core Intel Core i5, 16GB 3733 MHz LPDDR4X on MacOS Ventura 13.1.
- May not work with M1 and M2 chips
- Running the `vagrant up` command will create a folder named .vagrant in the directory where it's run.
- Give no more RAM to the VM than it needs. That is totally guesswork. I recommend this, because there is no such thing as memory balooning on a Mac, so your VM will eat all the RAM you assigned to it, regardless of its workload.

## Not ready for SSH problem

You may encounter an error like this after running `vagrant up` while nfs server is being installed:

```
The provider for this Vagrant-managed machine is reporting that it is not yet ready for SSH...
```

This is a rare transient error. Run`vagrant up`  again and then `vagrant reload`. This should resolve this problem.

## Uses NFS version 3 with UDP.

- NFS with UDP is faster, but may have trouble handling files larger than 4GB.
- NFS version 3 is less secure than NFS version 4.

# Usage
## Walkthrough

Install vagrant: https://developer.hashicorp.com/vagrant/docs/installation

Install virtualbox: https://www.virtualbox.org/wiki/Downloads

Download the Vagrant box. Run exactly this command, because newer versions of Ubuntu do not support NFSv3:

`vagrant box add --box-version 202303.13.0 bento/ubuntu-22.04 --provider virtualbox` 

Install the vagrant nfs guest plugin:

`vagrant plugin install vagrant-nfs_guest`

Install the vagrant-vbguest plugin:

`vagrant plugin install vagrant-vbguest`

Edit the `config.vm.synced_folder` directive in the Vagrantfile to have the shared folder where you want it. When provisioning the VM, the contents of the mounted folder on the host <mark style="background: #ea3323">WILL BE LOST</mark>, so set an empty directory as the synced folder.

The name of the folder defined for the VM will be the name of the folder on the host as well.

Provision and start the VM: `vagrant up`

After the VM is ready copy the files you want to be available in the VM into synced folder.

SSH to the VM: `vagrant ssh` After that cd to folder shared with the host machine and you can run your lando commands or whatever you want.

## Tips

Save the VM's state and stop it (poweroff): `vagrant halt`

Start a halted VM: `vagrant up`

Pause the VM: `vagrant suspend`

Unpause the VM: `vagrant resume`

Reload the VM after changing its configuration (the `--provision` flag may be needed): `vagrant reload`

Checking the status of your Vagrant machines can be buggy, so use the `vagrant global-status --prune` command to have up-to-date information of all of your Vagrant machines. You can also use Virtualbox or Parallels for that.

<!-- TODO: -->
It is possible to reach docker containers running in Vagrant. You just have to set up port forwarding in your Vagrantfile with the `config.vm.network "forwarded_port", guest: 80, host: 8080`. Forwarded ports can be reached both at 0.0.0.0 and the VM's IP address - configured to be 192.168.56.4 by the Vagrantfile in this repo.
At the time of writing, getting https to work with a Docker container in the VM and the host machine as a client is in progress.

# Possible improvements

- Not enough memory in VM => use swapfile: https://github.com/ianmiell/vagrant-swapfile/blob/master/vagrant-swapfile.sh
- Using Parallels instead of VirtualBox may yield some improvements
- Using vagrant box which needs less resources than Ubuntu. Note that the nfs guest plugin supports debian based distros only. Its forks may support other distros.

# Notes on testing

I tested this solution on two workloads; one was a very large private repository, which I am not allowed to share, the other is:
[https://github.com/kburakozdemir/lando-drupal9-recipe](https://github.com/kburakozdemir/lando-drupal9-recipe)

You can use this to verify that you have set up everything correctly, because dockerized PHP applications are exceptionally slow in Docker on Mac.
I was going to paste detailed test results here, but unfortunately, I forgot to save the logs. For very large projects this workaround yielded 90% faster lando commands than doing the same with Docker Desktop.
