These instructions will walk you through the process of migrating a Linode VPS to your own Xen host running on a Hetzner dedicated server. 

**For the background to these instructions, please [see the README](README.md).**

In these instructions, I:

1. Set up a Hetzner dedicated server as a Xen Dom0 host, with Ubuntu 12.04.
2. Migrate an Ubuntu (10.04+) Linode to run as a DomU guest on that host, with its own IP, using bridged networking.
3. Set up a fresh DomU guest, with a host-local IP using NAT.
4. Configure Varnish on the Dom0 to make the NAT'd DomU visible to the world on HTTP.

There are, of course, a number of ways to set up networking with Xen. If you want each guest machine to have its own IP, as a Linode does, bridged networking is the way to go. Hetzner will sell you up to 4 IPs per server, which means one host can support 3 bridged guest machines (with one IP assigned to the Dom0 host). You can alternatively buy a subnet with more addresses, but this comes with a rather hefty price tag (and, if I reacll correctly, may not work the same way, as you don't get MAC addresses for the subnet IPs).

If you want more than 3 guest machines on one host, you can combine NAT with the bridged setup. You can use NAT for any guest machine that's accessible only over HTTP, by installing a proxy (e.g. Varnish) on the Dom0 host to provide access to such DomU guests. That won't work, for example, if you need non-SNI HTTPS access to the guest. But sometimes you might not.

Caveats
-------

* If you follow these instructions, I take absolutely no responsibility and accept no liability for bad things happening. Obviously. But I sincerely hope they don't.

* I've produced this document as a GitHub repo to allow for corrections/enhancements via pull requests, and for forking (e.g. for different hosting providers and distros). Fork/request away! However, I'm not likely to have the time to do a lot of troubleshooting, so _I don't promise to respond to any issues_.

* This isn't intended to be a tutorial about best security practices in setting up a Linux server. For example, I won't be setting up public key SSH authorization, `fail2ban`, and so on. And you might well disagree with me on other matters (e.g. disk partitioning).

* I'm not a sysadmin. I'm a developer who does his own sysadmin. If I'm doing something egregious, feel free to let me know.


Dom0 (host) setup
-----------------

First off, let's set up our dedicated Hetzner machine. I'm going to use Ubuntu 12.04 (LTS), which supports operation as a Xen Dom0 (virtualisation host).

    ssh root@nnn.nnn.nnn.nnn  

    passwd  # change default password
    installimage

    # U(buntu)
    # 1204-precise-minimal

The `installimage` program then opens us an editor in which we can modify the default settings. The main thing I change here is that I allocate most disk space to an LVM partition for flexibility in future.

    SWRAID 1
    SWRAIDLEVEL 1
    # note: installimage also sets boot_degraded to true, as desired
    
    HOSTNAME gmxentest

    # original disk setup:
    # PART swap swap 2G
    # PART /boot ext3 512M
    # PART / ext4 all
    
    # changed to:
    PART  /boot ext3  512M
    PART  lvm   vg0   all
    LV    vg0   swap  swap swap    2G
    LV    vg0   root  /    ext4   10G
    
With those changes made, we can reboot the machine.

    reboot

On the local machine we need to delete the RSA key for nnn.nnn.nnn.nnn, since the Ubuntu installation has changed it and that will make SSH complain. Then we log in again.

    nano ~/.ssh/known_hosts  # delete old RSA key for nnn.nnn.nnn.nnn

    ssh root@nnn.nnn.nnn.nnn

Hetzner by default retrieves packages from its own Ubuntu mirrors, saving you a little paid bandwidth. However, I found these mirrors were broken, so I replace them with the canonical (German) versions.

    cp /etc/apt/sources.list /etc/apt/sources.list.`date +%F`

    echo 'deb http://de.archive.ubuntu.com/ubuntu/ precise main restricted universe multiverse 
    deb-src http://de.archive.ubuntu.com/ubuntu/ precise main restricted universe multiverse 
    deb http://de.archive.ubuntu.com/ubuntu/ precise-security main restricted universe multiverse 
    deb http://de.archive.ubuntu.com/ubuntu/ precise-updates main restricted universe multiverse 
    deb-src http://de.archive.ubuntu.com/ubuntu/ precise-security main restricted universe multiverse 
    deb-src http://de.archive.ubuntu.com/ubuntu/ precise-updates main restricted universe multiverse' > /etc/apt/sources.list

Now I upgrade Ubuntu and install the basic packages we need for Xen. Plus Byobu, which I can't live without.

    aptitude update
    aptitude safe-upgrade
    aptitude install byobu xen-hypervisor-amd64 xen-tools bridge-utils

Some housekeeping next: I set up a new administrative user, and slightly harden up `sshd` — including switching to a non-default port and preventing root log-ins.

    adduser george --ingroup sudo

    sed -r \
    -e 's/^Port 22$/Port 2200/' \
    -e 's/^PermitRootLogin yes$/PermitRootLogin no/' \
    -e 's/^X11Forwarding yes$/X11Forwarding no/' \
    -e 's/^UsePAM yes$/UsePAM no/' \
    -i.`date +%F` /etc/ssh/sshd_config

    reboot

Now I log back in as the new user, set up Byobu, and get `iptables` up and running. These basic `iptables` rules let anyone access the machine via HTTP(S), but only permit SSH access from a few trusted IP addresses.

    ssh -p 2200 george@nnn.nnn.nnn.nnn

    byobu  # Fn-F9 to always use

    sudo su

    # iptables -F  # no: don't flush Xen's default forwarding rules
    iptables -A FORWARD -j DROP
    iptables -A OUTPUT -j ACCEPT

    iptables -A INPUT -i lo -j ACCEPT
    iptables -A INPUT ! -i lo -d 127.0.0.0/8 -j DROP
    iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
    iptables -A INPUT -p tcp --dport http -j ACCEPT
    iptables -A INPUT -p tcp --dport https -j ACCEPT

    iptables -N specialips
    iptables -A specialips -s xxx.xxx.xxx.xxx -j RETURN  # a trusted IP address
    iptables -A specialips -s yyy.yyy.yyy.yyy -j RETURN  # ditto
    iptables -A specialips -j DROP

    iptables -A INPUT -j specialips
    iptables -A INPUT -p tcp --dport 2200 -j ACCEPT
    iptables -A INPUT -j DROP

    iptables-save > /etc/iptables.rules

A few more tweaks: make sure relatime is set (though I think this may be the default), and set up mail + `mdadm` monitoring for disk failures.

    cp /etc/fstab /etc/fstab.`date +%F`

    nano /etc/fstab  # change defaults to defaults,relatime on /dev/vg0/root

    aptitude install postfix  # Internet site profile
    
    echo 'george: me@example.com
    root: me@example.com' >> /etc/aliases

    newaliases

Test that a RAID failure does result in a mail notification.

    mdadm --manage --set-faulty /dev/md0 /dev/sdb1

    mdadm /dev/md0 -r /dev/sdb1   # remove disk
    mdadm /dev/md0 -a /dev/sdb1   # re-add disk

Now set up `smartd` for other disk status warnings.

    sed -e 's/^#start_smartd=yes/start_smartd=yes/' -i.`date +%F` /etc/default/smartmontools

    cp /etc/smartd.conf /etc/smartd.conf.`date +%F`
    
    echo '
    /dev/sda -a -o on -S on -s (S/../.././02|L/../../3/03) -m root -M test
    /dev/sdb -a -o on -S on -s (S/../.././04|L/../../7/03) -m root -M test
    ' > /etc/smartd.conf

    /etc/init.d/smartmontools restart

Now we're ready to get going with Xen. I use the deprecated `xm` toolstack, because I couldn't get the new `xl` toolstack to work. It might just be me, but I'm not sure `xl` is ready for primetime.

I also switch the Xen kernel to be the default, and set the Dom0 (host) machine to use a constant 1GB of RAM.

    sed -i.`date +%F` 's/TOOLSTACK=.*\+/TOOLSTACK="xm"/' /etc/default/xen

    sed -i.`date +%F` 's/^GRUB_DEFAULT=.*/GRUB_DEFAULT="Xen 4.1-amd64"/' /etc/default/grub

    echo 'GRUB_CMDLINE_XEN="dom0_mem=1024M"' >> /etc/default/grub

    update-grub

Now I set up the network. As I recall, the Xen documentation claims you don't need to manually set up a bridge, but this turns out to be a lie.

Obviously you'll use the address, netmask and gateway you got from Hetzner here.

    cp /etc/network/interfaces /etc/network/interfaces.`date +%F`

    echo 'auto lo
    iface lo inet loopback

    auto xenbr0
    iface xenbr0 inet static
     address nnn.nnn.nnn.nnn
     netmask mmm.mmm.mmm.mmm
     gateway ggg.ggg.ggg.ggg
     bridge_ports eth0
     bridge_stp off
     bridge_fd 1
     bridge_hello 2
     bridge_maxage 12

    pre-up iptables-restore < /etc/iptables.rules' > /etc/network/interfaces

Hetzner suggests proxy_arp (http://wiki.hetzner.de/index.php/Zusaetzliche_IP-Adressen/en). Xen suggests disabling iptables on the bridge (http://wiki.xen.org/wiki/Network_Configuration_Examples_%28Xen_4.1%2B%29).
    
    echo '
    net.ipv4.conf.default.proxy_arp = 1
    net.bridge.bridge-nf-call-ip6tables = 0
    net.bridge.bridge-nf-call-iptables = 0
    net.bridge.bridge-nf-call-arptables = 0' >> /etc/sysctl.conf

Now I get xen to run the appropriate scripts for bridged networking. I also change Dom0 memory settings.

    nano /etc/xen/xend-config.sxp

    # uncomment:
    network-script network-bridge
    vif-script vif-bridge

    # change:
    (dom0-min-mem 1024)
    (enable-dom0-ballooning no)

I prefer to prevent DomU (guest) hibernation, because it doesn't seem to work reliably.

    sed -r \
    -e 's/^XENDOMAINS_RESTORE=.*/XENDOMAINS_RESTORE=false/' \
    -e 's/^XENDOMAINS_SAVE=.*/XENDOMAINS_SAVE=""/' \
    -i.`date +%F` /etc/default/xendomains

    reboot


Migrate Linode VPS to DomU (guest)
----------------------------------

Now it's time to migrate our Linode. 

Note that for this guest machine I'm using bridged networking, which means requesting an extra IP address from Hetzner. I will refer to this address as zzz.zzz.zzz.zzz. You also need to assign your DomU the correct MAC address provided by Hetzner for that IP, which I'll refer to as zz:zz:zz:zz:zz:zz.

Using Linode's web-based admin, I shut the Linode down and boot back up in Rescue Mode.

In the Linode's AJAX console I set a root password and start `sshd`.

    passwd
    /etc/init.d/ssh start

Back at Hetzner, I retrieve the disk image over SSH.
    
    sudo su
    ssh -C root@my.linode "dd if=/dev/xvda bs=1M" | dd of=my-linode.img  # -C = compression

Now I create a logical volume of the right size for the Linode image, and copy the image data into it.
    
    lvcreate -L `wc -c my-linode.img`b -n my-domu-root vg0
    dd if=my-linode.img of=/dev/vg0/my-domu-root bs=1M

Optionally, we can give ourselves some more disk space on Hetzner.

    lvextend -L+1G /dev/vg0/my-domu-root
    e2fsck -f /dev/vg0/my-domu-root
    resize2fs -p /dev/vg0/my-domu-root

Then I create an LVM swap space volume.

    lvcreate -L 512M -n my-domu-swap vg0

    mkswap -f /dev/vg0/my-domu-swap

Now to create a Xen config file for the new DomU. 

    echo "kernel = '/vmlinuz'
    ramdisk = '/initrd.img'
    vcpus = '1'
    memory = '512'
    root = '/dev/xvda2 ro'
    disk = [
      'phy:/dev/vg0/my-domu-root,xvda2,w',
      'phy:/dev/vg0/my-domu-swap,xvda1,w'
    ]
    hostname = 'mydomu'
    name = 'my-domu'
    vif = [ 'script=vif-bridge,ip=zzz.zzz.zzz.zz,mac=zz:zz:zz:zz:zz:zz' ]
    on_poweroff = 'destroy'
    on_reboot = 'restart'
    on_crash = 'restart'" > /etc/xen/my-domu.cfg

Make this DomU start when the Dom0 boots.

    mkdir -p /etc/xen/auto
    ln -s /etc/xen/my-domu.cfg /etc/xen/auto/my-domu.cfg

Now let's try to start and attach to the new DomU.

    mkdir /var/lib/xen
    xm create /etc/xen/my-domu.cfg
    xm console my-domu

Assuming we now have a shell in our migrated Linode DomU, we proceed to tweak its network configuration.

Note that we use the same netmask and gateway here as for the Dom0 host.

Remember to delete or amend the `pre-up` statement if this machine isn't using `iptables` or has a different rules file.

    cp /etc/network/interfaces /etc/network/interfaces.`date +%F`

    echo 'auto lo
    iface lo inet loopback

    auto eth0

    iface eth0 inet static
     address zzz.zzz.zzz.zz
     netmask mmm.mmm.mmm.mmm
     gateway ggg.ggg.ggg.gg

    pre-up iptables-restore < /etc/iptables.rules' > /etc/network/interfaces

Point our guest to the Hetzner DNS servers.

    cp /etc/resolv.conf /etc/resolv.conf.`date +%F`
    echo 'nameserver 213.133.98.98
    nameserver 213.133.99.99
    nameserver 213.133.100.100
    ' > /etc/resolv.conf

Now I install a kernel on the host machine. On Linode, kernels are provided externally, and we're currently running this DomU with the host Dom0's kernel. But it seems neater and more robust to give this machine its own kernel. In a minute, we'll set up `pvgrub` to allow Xen to boot it.

    aptitude install linux-generic-pae linux-headers-generic-pae grub

    update-grub  # say yes to creating menu.lst: it's the only reason to install grub

You might want to edit various config files (e.g. web server `Listen` directives) with the machine's new IP at this point.

    nano /opt/nginx/conf/nginx.conf

I decide to upgrade the guest's filesystem to ext4 too, and do the following in anticipation.

    nano /etc/fstab  # change ext3 to ext4

Back on Hetzner now, I shut down the guest to do some more tweaking.

    xm shutdown my-domu

Convert the guest filesystem to ext4.

    tune2fs -O extents,uninit_bg,dir_index /dev/vg0/my-domu-root
    fsck -fp /dev/vg0/my-domu-root

Switch to the guest kernel we installed, and restart.

    nano /etc/xen/my-domu.cfg

    # delete 'kernel' + 'ramdisk' lines

    # then add:
    # bootloader = '/usr/lib/xen-4.1/bin/pygrub'

    xm create /etc/xen/my-domu.cfg

That's it! Though you might also want to set up LVM snapshot backups on the Dom0 host.


New DomU guest with NAT
-----------------------

Now I set up a fresh DomU guest machine, using NAT networking. Since we've now got a bridged/NAT hybrid setup, this involves a custom networking script.

On the Dom0 host:
    
    sudo su
    nano /etc/xen/scripts/vif-nat-gm 

And paste in:

    #!/bin/bash

    dir=$(dirname "$0")
    . "$dir/vif-common.sh"

    # router IP is derived by arbitrarily adding 127 to VM IP
    router_ip=$(echo $(echo "$ip" | awk -F. '{print $1"."$2"."$3"."$4 + 127}'))  
    vif_ip=`echo ${ip} | awk -F/ '{print $1}'`  # removes /NN if present

    case "$command" in
      online)
        if ip route | grep -q "dev ${dev}"
        then
          log debug "${dev} already up"
          exit 0
        fi
        ip link set "${dev}" up arp on
        ip addr add "$router_ip" dev "${dev}"
        ip route add "$vif_ip" dev "${dev}" src "$router_ip"
        echo 1 > /proc/sys/net/ipv4/conf/${dev}/proxy_arp
        echo 1 > /proc/sys/net/ipv4/ip_forward
        ;;
      offline)
        do_without_error ifconfig "${dev}" down
        ;;
    esac

    if [ "$command" == "online" ]
    then
      c="-I"   
    else
      c="-D"
    fi

    iptables -t nat "$c" POSTROUTING -o xenbr0 -s "$vif_ip" -j MASQUERADE
    iptables "$c" FORWARD -i "${dev}" -o xenbr0 -s "$vif_ip" -j ACCEPT
    iptables "$c" FORWARD -i xenbr0 -o "${dev}" -m state --state RELATED,ESTABLISHED -j ACCEPT

    log debug "Successful vif-nat-gm $command for ${dev}."
    if [ "$command" = "online" ]
    then
      success
    fi

Fix permissions on this file.

    chmod +x /etc/xen/scripts/vif-nat-gm

Now I create blank LVM volumes for root and swap filesystems.

    lvcreate -L 10G -n fresh-domu-root vg0
    mkfs.ext4 /dev/vg0/fresh-domu-root

    lvcreate -L  2G -n fresh-domu-swap vg0
    mkswap -f /dev/vg0/fresh-domu-swap

Retrieve Ubuntu 12.04 installer.

    mkdir -p /var/lib/xen/images/ubuntu-netboot
    cd /var/lib/xen/images/ubuntu-netboot
    wget http://de.archive.ubuntu.com/ubuntu/dists/precise/main/installer-amd64/current/images/netboot/xen/initrd.gz
    wget http://de.archive.ubuntu.com/ubuntu/dists/precise/main/installer-amd64/current/images/netboot/xen/vmlinuz

Now I create a Xen config file for the new guest. I use various `extra` installer parameters to get things set up correctly.

This being the first NAT'd DomU, I pick 10.0.0.1 as the IP. The custom script we're using assumes a gateway IP that's the guest IP + 127, so I use 10.0.0.128 as the gateway.

    echo "kernel = '/var/lib/xen/images/ubuntu-netboot/vmlinuz'
    ramdisk = '/var/lib/xen/images/ubuntu-netboot/initrd.gz'
    extra = 'debian-installer/exit/always_halt=true debian-installer/locale=en_GB netcfg/get_ipaddress=10.0.0.1 netcfg/get_netmask=255.255.255.0 netcfg/get_gateway=10.0.0.128 netcfg/get_hostname=freshdomu netcfg/get_domain=fresh-domu.com netcfg/get_nameservers=213.133.99.99 -- console=hvc0'
    vcpus = '2'
    memory = '2048'
    root = '/dev/xvda2 ro'
    disk = [
      'phy:/dev/vg0/fresh-domu-root,xvda2,w',
      'phy:/dev/vg0/fresh-domu-swap,xvda1,w'
    ]
    hostname = 'freshdomu'
    name = 'fresh-domu'
    vif = [ 'script=vif-nat-gm,ip=10.0.0.1' ]
    on_poweroff = 'destroy'
    on_reboot = 'restart'
    on_crash = 'restart'" > /etc/xen/fresh-domu.cfg

Now I start up the DomU.

    xm create /etc/xen/fresh-domu.cfg -c

Ubuntu setup proceeds. I set relatime on /. Turn on auto updates. 

GRUB installation fails for me at this point, so I update the Xen config to boot with the Dom0 kernel temporarily.

    xm shutdown fresh-domu

    nano /etc/xen/fresh-domu.cfg

    # delete 'extra' line

    # then add:
    # kernel = '/vmlinuz'
    # ramdisk = '/initrd.img'

I restart the DomU (the `-c` option attaches to the console immediately).

    xm create /etc/xen/fresh-domu.cfg -c

On the DomU, try again to install GRUB.
    
    sudo su

    aptitude install grub
    update-grub  # say yes to menu.lst
    shutdown -h now

Back on the Dom0, we switch to using the DomU's own kernel.

    nano /etc/xen/fresh-domu.cfg

    # delete 'kernel' + 'ramdisk' lines

    # then add: 
    # bootloader = '/usr/lib/xen-4.1/bin/pygrub'

Make this machine start automatically too.

    mkdir -p /etc/xen/auto
    ln -s /etc/xen/fresh-domu.cfg /etc/xen/auto/fresh-domu.cfg

And restart it.

    xm create /etc/xen/fresh-domu.cfg


Set up Varnish to access NAT DomU over HTTP
-------------------------------------------

The fresh DomU is accessible on 10.0.0.1 from the Dom0, but nobody else can see it. Assuming this is a web server, we can make it accessible to the world with an HTTP proxy.

(Note: if this was the only guest using port 80 on the Dom0, we could just forward all port 80 traffic. However, I plan to set up multiple web server DomUs, so this HTTP proxy solution is required).

Here I show how we can use Varnish for this purpose.

We're working on the Dom0 now.
    
    sudo su

    aptitude install varnish

    cp /etc/varnish/default.vcl /etc/varnish/default.vcl.`date +%F`

    echo '
    backend freshdomu {
      .host = "10.0.0.1";
      .port = "80";
    }
    sub vcl_recv {
      if    (req.http.host ~ "(?i)^(www\.)?fresh-domu-vhost1\.net$") { set req.backend = freshdomu; } 
      elsif (req.http.host ~ "(?i)^(www\.)?fresh-domu-vhost2\.org$") { set req.backend = freshdomu; }
      else { error 404 "Unknown virtual host"; }
    }
    ' > /etc/varnish/default.vcl

    sed -e 's/^DAEMON_OPTS="-a :6081/DAEMON_OPTS="-a :80/' -i.`date +%F` /etc/default/varnish

    /etc/init.d/varnish restart

If and when I add more web-serving DomUs, I can set them up as additional backends, and add rules to match the relevant virtual host names.

I hope you found this useful.
