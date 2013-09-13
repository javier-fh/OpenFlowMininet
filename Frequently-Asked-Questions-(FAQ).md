==Frequently Asked Questions (FAQ)==

===How can you convert a VirtualBox VM/.vdi image to a VM or .vmdk disk image usable by VMware?===

If you have VMware installed, you may find that it is advantageous to run the VM under VMware rather than VirtualBox.

'''For VMware Player on Windows''', use the "Export Appliance..." menu command from inside VirtualBox. Make sure you select the "Write Manifest file" option. This will create a .ova virtual machine archive which can be imported into VMware.

To import the .ova into VMware Player on Windows, open it using the "Open..." menu command.

'''For VMware Fusion on the Mac''', it's slightly more complicated, but not terribly so. Use the "Export Appliance..." menu command from inside VirtualBox, making sure to save an .ovf (and .vmdk) file by pressing "Choose..." and selecting "Open Virtualization Format (*.ovf)" from the "Files of type:" pop-up menu. This will create a .ovf file, which you can ignore (or import using VMware's ovf tool if you have it installed), and a .vmdk disk image, which is exactly what you need.

(If you create an .ova file by mistake, you can either try again or rename the .ova file to a .tar file and extract it to get the .vmdk image.)

Next, create a new VM in VMware Fusion, delete its default hard disk (if any), and add a new hard disk, specifying the .vmdk file as the existing disk image to use. Once the VM has a single hard drive specified as the .vmdk file, it should be able to boot and run.

'''For VMware player on Ubuntu''', you should be able to use the above approach that works for Windows, but I haven't tested it yet.

Alternately, you can create a .vmdk image from the .vdi file by using <code>qemu-img</code> from the <code>qemu</code> package; for example:

 qemu-img convert -f vdi tutorial.vdi -O vmdk tutorial.vmdk

You can then attach it to a new VM as described in the VMware Fusion instructions.

=== How does one scroll an xterm?===
Start it with the flag -sb 500 to store 500 lines of output, e.g.

 xterm -sb 500 &

then use a scroll wheel, trackpad scrolling, or the middle mouse button to scroll with the quaint Athena scroll widget (i.e. grey bar on the left side of the window.)

If you want a more advanced terminal, you might wish to try installing gnome-terminal, e.g.

 sudo apt-get update
 sudo apt-get install gnome-terminal
 gnome-terminal &

=== Why can't I ssh into my VirtualBox VM? ===

You may not be able to connect to a VirtualBox VM that only has NAT networking enabled. Make sure that you have followed the instructions above and '''configured host-only networking on at least one interface in the VM settings''', and make sure that the '''host-only interface''' is configured and that you are connecting to its correct IP address.

If this does not work, or for more advanced setup, see below.

=== I can't start WireShark or xterm - help! ===

You are probably getting an error like "cannot open display."

This probably means that you have not successfully connected to the VM with ssh and X11 forwarding enabled.

Make sure that you have carefully followed our instructions above. If you are on Windows, make sure that

* You have '''installed''' Xming and PuTTY
* You have '''started up Xming''' (important!) and it is '''still running'''
* You have enabled X11 forwarding
* You have verified that you can ssh in with X11 forwarding and start up an xterm

'''If you cannot start up an xterm, you will not be able to run wireshark!'''
