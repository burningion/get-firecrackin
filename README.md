# Firecracker Notes

Building a filesystem from a Docker image:

https://github.com/firecracker-microvm/firecracker/blob/master/docs/rootfs-and-kernel-setup.md

First, create an empty file, 500 megabytes big:

`dd if=/dev/zero of=rootfs.ext4 bs=1M count=500`

Create an EXT4 file system on the file you created:

`mkfs.ext4 rootfs.ext4`

You now have an empty EXT4 image in rootfs.ext4, so let's prepare to populate it. First, you'll need to mount this new file system, so you can easily access its contents:


```
mkdir /tmp/my-rootfs
sudo mount rootfs.ext4 /tmp/my-rootfs
```

The minimal init system would be just an ELF binary, placed at /sbin/init. The final step in the Linux boot process executes /sbin/init and expects it to never exit. More complex init systems build on top of this, providing service configuration files, startup / shutdown scripts for various services, and many other features.

For the sake of simplicity, let's set up an Alpine-based rootfs, with OpenRC as an init system. To that end, we'll use the official Docker image for Python, which extends Alpine. 

Looking at the [Python Docker Hub page](https://hub.docker.com/_/python), it looks like we want the tag `3.8.1-alpine3.11`.


Let's start the Alpine container, mounting the EXT4 image created earlier, to /my-rootfs:

```
docker run -it --rm -v /tmp/my-rootfs:/my-rootfs python:3.8.1-alpine3.11 sh
```

Then, inside the container, install the OpenRC init system, and some basic tools:

```
apk add openrc
apk add util-linux
```

And set up userspace init (still inside the container shell):

```
# Set up a login terminal on the serial console (ttyS0):
ln -s agetty /etc/init.d/agetty.ttyS0
echo ttyS0 > /etc/securetty
rc-update add agetty.ttyS0 default

# Make sure special file systems are mounted on boot:
rc-update add devfs boot
rc-update add procfs boot
rc-update add sysfs boot

# Then, copy the newly configured system to the rootfs image:
for d in bin etc lib root sbin usr; do tar c "$d" | tar x -C /my-rootfs; done
for dir in dev proc run sys var; do mkdir /my-rootfs/${dir}; done
for dir in cache run; do mkdir /my-rootfs/var/${dir}; done
mkdir /my-rootfs/var/cache/apk
```

Finally, we need to make sure we automatically login as the root user. 

```
vi /my-rootfs/etc/init.d/agetty.ttyS0
```

At the line with `command_args_foreground`, add "-a root" in front of the rest of the commands. This logs in as root automatically. (Do a `man agetty` in order to see more options here.) 

All done now, exit docker shell

```
exit
```

Finally, unmount your rootfs image:

```
sudo umount /tmp/my-rootfs
```

Now, we should be able to start up the filesystem using the included kernel. To do that, we need to first write a configuration file.

# Writing A Configuration File for Firecracker

The configuration file for Firecracker is relatively simple. At a minimum, we specify a kernel to boot, flags for the kernel, and a filesystem to mount. 

Optionally, we can add the number of CPUs and the amount of memory we'd like to be available for our microVM:

```json
{
  "boot-source": {
    "kernel_image_path": "hwrandom-vsock.vmlinux",
    "boot_args": "console=ttyS0 noapic reboot=k panic=1 pci=off random.trust_cpu=on nomodules rw"
  },
  "drives": [
    {
      "drive_id": "rootfs",
      "path_on_host": "rootfs.ext4",
      "is_root_device": true,
      "is_read_only": false
    }
  ],
  "machine-config": {
    "vcpu_count": 2,
    "mem_size_mib": 1024,
    "ht_enabled": false
  }
}
```

Save this JSON file as `first-config.json`, and run it using the following command:

```bash
firecracker --config-file first-config.json
```

With that, we should spin up our microVM, and be dropped right into an `sh` instance.

Try running `python3` to verify everything worked. You can exit Python3 by typing in `exit()`.

# Setting Up the Network On the Host

If we try running `ping google.com` inside of our new microVM, nothing happens. That's because we haven't set up any networking yet.

We'll set up a `tap` interface on our underlying host, and then make that available for our microVM. So let's begin on the host.

You can exit the microVM by typing `reboot`.

Let's add the `tap` interface on the host machine:

```bash
sudo ip tuntap add tap0 mode tap # user $(id -u) group $(id -g)
sudo ip addr add 172.20.0.1/24 dev tap0
sudo ip link set tap0 up
```

Set your main interface device. Check to make sure you've got the right adapter with `ifconfig` command. It should have an IP address assigned to it, if you've got internet on the host machine.

```
DEVICE_NAME=eth0
```

Next, we need to set up iptables rules to enable packet forwarding:

```
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
sudo iptables -t nat -A POSTROUTING -o $DEVICE_NAME -j MASQUERADE
sudo iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i tap0 -o $DEVICE_NAME -j ACCEPT
```

With this, we can grab our MAC address for the `tap0` device, and get ready to boot Firecracker with it:

```
MAC="$(cat /sys/class/net/tap0/address)"
echo $MAC
```

Great! Now that we've got a mac address, we need to add it to our `first_config.json`:

```json
    "network-interfaces": [{
        "iface_id": "aa:bb:cc:dd:ee:ff",
        "host_dev_name": "tap0"
    }],
```

Make sure to replace the `iface_id` with the actual MAC address on your machine. With that change, we can bring back up the VM:

```bash
firecracker --config-file first-config.json
```

# Enable Networking on the VM

In the VM, bring up the network:

```
ifconfig eth0 up && ip addr add dev eth0 172.20.0.6/24
ip route add default via 172.20.0.1
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

With this, we should be able to ping Google, ensuring we've got a running system:

```
ping google.com
```

# Enable Networking on Startup

Great! We don't want to have to do this every time we run the machine. So let's first automatically login as the root user on our machine:

Edit the `/etc/init.d/networking` file, and replace the `start()` function with the following:

```
start() {
        ifconfig eth0 up
        ip addr add dev eth0 172.20.0.6/24
        ip route add default via 172.20.0.1
        return 0
}
```

This brings up the ethernet on startup. Now if we issue a `reboot` in the shell, we can see it all spin up.

:neat:

# Ensuring Adequate Entropy

Trying to start Python, I get an error about entropy, and it just hangs. Can't run a server, and I can't run `pip`.

We need to add entropy to the virtual machine! It starts with 1 bit of entropy available.

You check the entropy available on a virtual machine with:

```
cat /proc/sys/kernel/random/entropy_availk l.  
```

On Alpine, done with haveged

```
apk add haveged
haveged -w 1024
cat /proc/sys/kernel/random/entropy_avail
```

OR

Build the Linux kernel with `CONFIG_RANDOM_TRUST_CPU=y` built in.

Once that's done, we then need to add the `random.trust_cpu=on` flag to the `kernel-opts`, so that on startup our kernel knows to trust the cpu's random.

(That's what I did, and I kept the haveged running on startup.)

If it works, you should see:

```
random: crng done (trusting CPU's manufacturer)
```

In the startup of your kernel.



# VSocks

Needed to rebuild the Linux kernel for the machine all over again, this time enabling support for VirtIO, Vsock, and KVM.

This works, the command they give you doesn't:

```
curl --unix-socket /tmp/firecracker.socket -X PUT 'http://localhost/vsock'   -d '{
      "vsock_id": "1",
      "guest_cid": 3,
      "uds_path": "./root"
    }'
```
