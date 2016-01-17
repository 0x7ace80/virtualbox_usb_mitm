# virtualbox_usb_mitm

For some resone, I want to implement USB Man-In-The-Middle function. 
My idea is :

[Real USB Device]  <---> [USB-MITM] <---> [Host OS]

But I found it is not easy to achieve this purpose.

- For Windows, there used to be a Device Simulation Framework on Windows 7. https://msdn.microsoft.com/en-us/library/ff538304(v=vs.85).aspx As a part of Windows Driver Kit, it can simulate all kinds of USB device. The implementation could be:

  [Window DSF generate USB traffics] <---> [Host] 

  But this module was deprecated since Windows 8. 

- For Linux there is USB Gadget, which enable USB port works under device mode instead of host mode as it usually do. 

[Real USB Device] <----> [USB Host <---> USB Gadget] <---> [Final Host]

To use USB Gadget the USB controller should support USB OTG. For most desktop for laptop computers, the USB ports are always work under host mode, so it might not support USB OTG. And USB Gadget is usually not enable by default, user might need recompile the linux kernel to enable it.

- Some Raspberry-Pi-like-PC can be configured to be a USB-MITM. BeagleBone support USB Gadget by default. There is a USBProxy project based on BeagleBone: https://github.com/dominicgs/USBProxy
The only drawback is ...$35 cost is kinds of high...

  An alternative way to implement this function is utilizing vitual machine. We can try to hack the traffic from host OS to the client OS. 

  [Real USB Device] <---> [HOST OS and VirtualMachine] <---> [Client OS]

  VirtualBox is the only one open source virtual machine with a relatively large user base. And it seems like a good candidate for the purpose.
  
# VirtualBox compilation setup
VirtualBox itself is 32bit (althrough it can virtualize 64bit OS). So it is really wise to use 32bit OS to build VirtualBox, which avoid a lot of annoying compatibility issues.
VirtualBox can be build under Window, Linux and OSX, but Linux (I use Ubuntu14_32bit in this case) gives you the most convenience way to setup all the libraries VirtualBox needed. Refer to this link to setup the libraries and build VirtualBox:
https://www.virtualbox.org/wiki/Linux%20build%20instructions

To fetech the source code:
https://www.virtualbox.org/wiki/Downloads

GCC/G++-4.9 are needed.

sudo add-apt-repository ppc:ubuntu-toolchain-r/test && sudo apt-get update && sudo apt-get install gcc-4.9 g++-4.9 

And set gcc/g++-4.9 to be default compiler.

Now decompress the source code downloaded from VirtualBox.org and then enter the root directory. Build the VirtualBox following the instructions as the manual said. The configuration command I use is 

./configure --disable-hardening --target-arch=x86

It takes about 1hr and a half to complete (on my old 2010 Macbook and Yes, I took some time to install Ubuntu on it and setup its drivers). Have a cup of coffee and wait.

When the build completes, you might need to build and isntall and load the drivers VBox will use. 

But the tricky thing is: it turned out that the drivers were not working on my machine. I had to install a VBox release version(with the same revision number as the source code) which helps to install the drivers correctly.

Not the VBox you build can be used, and you can installed a windows 10 as client machine.

# VirtualBox Print debug message.
There are many verbosity level in VBOX source code. The one can print debug message under release build is:

LogRel((char* ...))

Eg. 

LogRel(("This is a test, Number %d, Name %s", 1, "Test"));

You can print message at any place. Note that too much print will slow down the virtual machine speed.

# USB Foundmantal Knowledge.

In order to hack the USB traffic, some knowledge on USB is needed.

All USB traffic in the driver will be carried in the form of URB (USB Request Block), including both the request and response. URB will contains some information about the endpoint, direction and data payload.

So don't surpise to see a lot of URB related structures and variables in the source code.

# Hack USB traffic
