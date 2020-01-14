# Firecracker Notes

Building from a Docker image:

https://github.com/firecracker-microvm/firecracker/blob/master/docs/rootfs-and-kernel-setup.md


`dd if=/dev/zero of=/rootfs.ext4 bs=1M count=50`

Create an empty file system on the file you created:

`mkfs.ext4 rootfs.ext4`

You now have an empty EXT4 image in rootfs.ext4, so let's prepare to populate it. First, you'll need to mount this new file system, so you can easily access its contents:


```
mkdir /tmp/my-rootfs
sudo mount rootfs.ext4 /tmp/my-rootfs
```

The minimal init system would be just an ELF binary, placed at /sbin/init. The final step in the Linux boot process executes /sbin/init and expects it to never exit. More complex init systems build on top of this, providing service configuration files, startup / shutdown scripts for various services, and many other features.

For the sake of simplicity, let's set up an Alpine-based rootfs, with OpenRC as an init system. To that end, we'll use the official Docker image for Alpine Linux:



First, let's start the Alpine container, bind-mounting the EXT4 image created earlier, to /my-rootfs:

```
docker run -it --rm -v /tmp/my-rootfs:/my-rootfs alpine
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
vi /etc/init.d/agetty.ttyS0
```

At the line with `command_args_foreground`, add "-a root" in front of the rest of the commands. This logs in as root automatically. (Do a `man agetty` in order to see more options here.) 

# All done, exit docker shell

```
exit
```

Finally, unmount your rootfs image:

```
sudo umount /tmp/my-rootfs
```



# Setting Up the Network On the Host

Set up the network interface on the host!

```bash
sudo ip tuntap add tap0 mode tap # user $(id -u) group $(id -g)
sudo ip addr add 172.20.0.1/24 dev tap0
sudo ip link set tap0 up
```

Set your main interface device. If you have different name check it with ifconfig command

```
DEVICE_NAME=eth0
```

```
MAC="$(cat /sys/class/net/tap0/address)"
```

Start VM:

```
sudo firectl --kernel=hello-vmlinux.bin --root-drive=rootfs.ext4  --kernel-opts="console=ttyS0 noapic reboot=k panic=1 pci=off nomodules rw" --tap-device=tap0/$MAC
```

# Networking on the VM

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

# Taking Care of Startup

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

# Taking Care of Entropy

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
