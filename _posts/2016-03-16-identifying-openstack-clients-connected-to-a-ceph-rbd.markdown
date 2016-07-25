---
layout: post
title: Identifying OpenStack clients connected to a Ceph RBD
date: 2016-03-16 14:07
published: false
categories:
---

Another quick 'howto' for anyone else that's found themselves in this situation: It's possible for an orphaned VM to still have a client connection to an RBD volume, i.e a volume that Cinder thinks doesn't have any attachments.  Typical symptoms when attempting to delete such a volume are something along the lines of:

    {"host":"oscontrol1","message":"2016-02-17 15:46:22.009 2716 ERROR cinder.volume.manager [req-304feafa-fd47-4284-bdc2-1ad311cf5544 d1a60c7b4bbc427e8f2c3f99d0c2e6d2 18af493d49ba44a6a7bfe967b8afb829 - - -] Cannot delete volume 9cd1ee91-a84f-4983-a8ed-a0805d90f337: volume is busy","offset":2435351,"path":"/var/log/cinder/cinder-volume.log","shipper":"log-courier","type":"cinder_volume","@version":"1","@timestamp":"2016-02-17T15:46:22.009Z","tags":["cinder","oslofmt"],"timestamp":"2016-02-17 15:46:22.009","pid":"2716","loglevel":"ERROR","module":"cinder.volume.manager","logmessage":"[req-304feafa-fd47-4284-bdc2-1ad311cf5544 d1a60c7b4bbc427e8f2c3f99d0c2e6d2 18af493d49ba44a6a7bfe967b8afb829 - - -] Cannot delete volume 9cd1ee91-a84f-4983-a8ed-a0805d90f337: volume is busy","received_at":"2016-02-17T15:46:24.951Z"}

When this happens, it's actually impossible to determine from the OpenStack APIs exactly what has that volume still open as from Cinder's POV it has no attachments:

    ❯ cinder show 9cd1ee91-a84f-4983-a8ed-a0805d90f337 
    +---------------------------------------+-------------------------------------------+
    |                Property               |                  Value                    |
    +---------------------------------------+-------------------------------------------+
    |              attachments              |                    []                     |
    |           availability_zone           |                   nova                    |
    |                bootable               |                  false                    |
	 
    [ .. ]

    |                 status                |                available                  |
    |                user_id                |     a67ad346c9f34b9c9d40aa6476dd640c      |
    |              volume_type              |                 Ceph SSD                  |
    +---------------------------------------+-------------------------------------------+

You can try and delete it manually, but you'll probably see the following:

    root@ceph-mon-0:~# rbd -p cinder.volumes.flash rm volume-9cd1ee91-a84f-4983-a8ed-a0805d90f337 
	Removing image: 0% complete...failed. rbd: error: image still has watchers 
	This means the image is still open or the client using it crashed. Try again after closing/unmapping 
	it or waiting 30s for the crashed client to timeout.

We need to query Ceph to find out which client(s) (or 'watchers') have a connection open to this volume:

	root@ceph-mon-0:~# rbd -p cinder.volumes.flash info volume-9cd1ee91-a84f-4983-a8ed-a0805d90f337
	rbd image 'volume-9cd1ee91-a84f-4983-a8ed-a0805d90f337':
	    size 140 GB in 35840 objects
	    order 22 (4096 kB objects)
	    block_name_prefix: rbd_data.29b411037d7ceae
	    format: 2
	    features: layering, striping
	    flags:
	    stripe unit: 4096 kB
	    stripe count: 1

Using the 'block_name_prefix', change 'data' for 'header' and run the following command:

	root@ceph-mon-0:~# rados -p cinder.volumes.flash listwatchers rbd_header.29b411037d7ceae
	watcher=10.10.104.191:0/47647874 client.56949441 cookie=140172382491120

So the client in question is 10.10.104.191, which happens to map back to a
compute node ('aladdin' in this example).  From there, if we look at the
process table and grep for our volume UUID:

	nick@aladdin:~$ ps auxww | grep -i [9]cd1ee91-a84f-4983-a8ed-a0805d90f337
	libvirt+ 24187 31.3  7.0 32124696 18552904 ?   Sl   Feb10 3174:03 /usr/bin/qemu-system-x86_64 -name instance-0002f5c4 -S -machine pc-i440fx-trusty,accel=kvm,usb=off -cpu SandyBridge,+erms,+smep,+fsgsbase,+pdpe1gb,+rdrand,+f16c,+osxsave,+dca,+pcid,+pdcm,+xtpr,+tm2,+est,+smx,+vmx,+ds_cpl,+monitor,+dtes64,+pbe,+tm,+ht,+ss,+acpi,+ds,+vme -m 16384 -realtime mlock=off -smp 4,sockets=4,cores=1,threads=1 -uuid e6a96e04-733d-4b5a-b735-9cca9e6e6431 -smbios type=1,manufacturer=OpenStack Foundation,product=OpenStack Nova,version=2014.2.3,serial=00000000-0000-0000-0000-0cc47a486bdc,uuid=e6a96e04-733d-4b5a-b735-9cca9e6e6431 -no-user-config -nodefaults -chardev socket,id=charmonitor,path=/var/lib/libvirt/qemu/instance-0002f5c4.monitor,server,nowait -mon chardev=charmonitor,id=monitor,mode=control -rtc base=utc,driftfix=slew -global kvm-pit.lost_tick_policy=discard -no-hpet -no-shutdown -boot strict=on -device piix3-usb-uhci,id=usb,bus=pci.0,addr=0x1.0x2 -drive file=rbd:cinder.vms/e6a96e04-733d-4b5a-b735-9cca9e6e6431_disk:id=cinder:key=xxx==:auth_supported=cephx\;none:mon_host=10.10.104.184\:6789\;10.10.104.187\:6789\;10.10.104.214\:6789,if=none,id=drive-virtio-disk0,format=raw,cache=writeback -device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x4,drive=drive-virtio-disk0,id=virtio-disk0,bootindex=1 -drive file=rbd:cinder.volumes.flash/volume-9cd1ee91-a84f-4983-a8ed-a0805d90f337:id=cinder:key=xxx==:auth_supported=cephx\;none:mon_host=10.10.104.184\:6789\;10.10.104.187\:6789\;10.10.104.214\:6789,if=none,id=drive-virtio-disk1,format=raw,serial=9cd1ee91-a84f-4983-a8ed-a0805d90f337,cache=writeback -device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x6,drive=drive-virtio-disk1,id=virtio-disk1 -netdev tap,fd=30,id=hostnet0,vhost=on,vhostfd=33 -device virtio-net-pci,netdev=hostnet0,id=net0,mac=fa:16:3e:d7:ae:05,bus=pci.0,addr=0x3 -chardev file,id=charserial0,path=/var/lib/nova/instances/e6a96e04-733d-4b5a-b735-9cca9e6e6431/console.log -device isa-serial,chardev=charserial0,id=serial0 -chardev pty,id=charserial1 -device isa-serial,chardev=charserial1,id=serial1 -device usb-tablet,id=input0 -vnc 0.0.0.0:2 -k en-us -device cirrus-vga,id=video0,bus=pci.0,addr=0x2 -incoming fd:27 -device virtio-balloon-pci,id=balloon0,bus=pci.0,addr=0x5 -msg timestamp=on

From that we can determine the instance UUID, and double-check that with Nova
to see if it exists or not.  If it doesn't then that virtual machine must be
destroyed 'manually' using virsh:

	nick@aladdin:~$ sudo virsh list | grep instance-0002f5c4
	 19    instance-0002f5c4              running
	root@aladdin:~# virsh destroy instance-0002f5c4
	Domain instance-0002f5c4 destroyed

Now it should be possible to delete the original volume:

	❯ cinder delete 9cd1ee91-a84f-4983-a8ed-a0805d90f337
	Request to delete volume 9cd1ee91-a84f-4983-a8ed-a0805d90f337 has been accepted.

And then a few seconds later:

	❯ cinder show 9cd1ee91-a84f-4983-a8ed-a0805d90f337     
	ERROR: No volume with a name or ID of '9cd1ee91-a84f-4983-a8ed-a0805d90f337' exists.
