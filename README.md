# virtualbox_usb_mitm

Preview
=

For some resone, I want to implement USB Man-In-The-Middle function. 
My idea is :

`[Real USB Device]  <---> [USB-MITM] <---> [Host OS]`

But I found it is not easy to achieve this purpose.

- For Windows, there used to be a Device Simulation Framework on Windows 7. https://msdn.microsoft.com/en-us/library/ff538304(v=vs.85).aspx As a part of Windows Driver Kit, it can simulate all kinds of USB device. The implementation could be:

  `[Window DSF generate USB traffics] <---> [Host] `

  But this module was deprecated since Windows 8. 

- For Linux there is USB Gadget, which enable USB port works under device mode instead of host mode as it usually do. 

  `[Real USB Device] <----> [USB Host <---> USB Gadget] <---> [Final Host]`

  To use USB Gadget the USB controller should support USB OTG. For most desktop and laptop computers, the USB ports are always work under host mode, so it might not support USB OTG. And USB Gadget is usually not enable by default, user might need recompile the linux kernel to enable it.

- Some Raspberry-Pi-like-PC can be configured to be a USB-MITM. BeagleBone support USB Gadget by default. There is a USBProxy project based on BeagleBone: https://github.com/dominicgs/USBProxy
The only drawback is ...$35 cost is kinds of high...

  An alternative way to implement this function is utilizing virtual machine. We can try to hack the traffic from host OS to the client OS. 

  `[Real USB Device] <---> [HOST OS and VirtualMachine] <---> [Client OS]`
  
  There are some tickets in StackOverflow mentioned this approch but no one give detailed information unfortunately.
  
  VirtualBox is the only one open source virtual machine with a relatively large user base. And it seems like a good candidate for the purpose.
  
VirtualBox Compilation Setup
=

VirtualBox itself is 32bit (althrough it can virtualize 64bit OS). So it is really wise to use 32bit OS to build VirtualBox, which avoid a lot of annoying compatibility issues.

VirtualBox can be build under Window, Linux and OSX, but Linux (I use Ubuntu14_32bit in this case) gives you the most convenience way to setup all the libraries VirtualBox needed. Refer to this link to setup the libraries and build VirtualBox:
https://www.virtualbox.org/wiki/Linux%20build%20instructions

To fetech the source code:
https://www.virtualbox.org/wiki/Downloads

GCC/G++-4.9 are needed.

`sudo add-apt-repository ppc:ubuntu-toolchain-r/test && sudo apt-get update && sudo apt-get install gcc-4.9 g++-4.9`

And set gcc/g++-4.9 to be default compiler.

Now decompress the source code downloaded from VirtualBox.org and then enter the root directory. Build the VirtualBox following the instructions as the manual said. The configuration command I use is 

`./configure --disable-hardening --target-arch=x86`

It usually takes about 1hr to complete (on my old 2010 Macbook and Yes, I took some time to install Ubuntu on it and setup its drivers). Have a cup of coffee and wait.

When the build completes, you might need to build, install and load the drivers VBox will use. It can be done by execute command at `out/Linux.x86/release/bin`

`make && make install && make load`

But the `TRICKY` thing is: it turned out that the drivers were not working on my machine. If I executed the binary `./VirtualBox`, it always reported error and failed when it started to run the virtual machine.
I had to install a prebuild release version VBox (with the same revision number as the source code) in offical way which helps to install the drivers correctly.

Now the VBox you build can work, and you can install a windows 10 as client OS.

USB Foundmantal Knowledge
=

All USB traffics in the driver will be carried in the form of `URB` (USB Request Block), including both the request and response. URB will contain information like endpoint, direction and data payload.

So don't surpise to see a lot of URB related structures and variables in the source code.

In Linux, the System URB structure is defined in `/usr/include/linux/usbdevice_fs.h`

```c
struct usbdevfs_urb {
        unsigned char type; // Can be Control, Interrupt, Bulk or Isochronus, we usually only care Bulk transfer.
        unsigned char endpoint; // If the endpoint >= 0x80, this is INPUT package.
        int status;
        unsigned int flags;
        void *buffer; // Pointer of the data buffer
        int buffer_length; // Size of the data buffer
        int actual_length; // Data size
        int start_frame;
        int number_of_packets;
        int error_count;
        unsigned int signr;     /* signal to be sent on completion, or 0 if none should be sent. */
        void *usercontext;
        struct usbdevfs_iso_packet_desc iso_frame_desc[0];
};
```

Hack USB Traffic
=

VBox virtualized USB device and use the virtualized usb device to communicate with real USB device on host OS.

`[VirtualBox VirtualMachineMonitor] -> [VBox Device Model (Proxy)] -> [ HostOS VBox Driver] -> [Real USB Device]`

the model of USB device proxy located at:

`src/VBox/Devices/USB/linux/USBProxyDevice-linux.cpp`

The function this USB device model will call to issue USB request is:

`static DECLCALLBACK(int) usbProxyLinuxUrbQueue(PUSBPROXYDEV pProxyDev, PVUSBURB pUrb)`

and the function to handle response is :

`static DECLCALLBACK(PVUSBURB) usbProxyLinuxUrbReap(PUSBPROXYDEV pProxyDev, RTMSINTERVAL cMillies)`

The structure which contains the system URB is `pUrbLnx->pKUrb`. and `pUrbLnx` is internal URB format used by VBox.

E.g. You can modify the response data in the end of `usbProxyLinuxUrbReap` before "return" statement of cause.
```c
if (pUrbLnx->enmDir == VUSBDIRECTION_IN && pUrb->EndPt == 1) // This is input URB from endpoint 1.
{
  for (unsigned int i = 0; i < pUrbLnx->pKUrb.buffer_length; i++)
  {
      pUrbLnx->pKUrb.buffer[i] = i;
  }
}
```
Usually, there are many kinds of URB will go through `usbProxyLinuxUrbReap` function, a filter function should be used to make sure we modify the correct URB package. 

Debug
=

There are many verbosity level in VBox source code. The one can print debug message under release build is:

```c
LogRel((char* ...))
```
Eg. 
```c
LogRel(("This is a test, Number %d, Name %s", 1, "Test"));
```
You can print message at any place. Note that too much print will slow down the virtual machine speed.

USB traffic can also be logged under Pcap format in VBox:

`VBoxManage controlvm "VM name" usbattach "device uuid|address" --capturefile "filename"`
