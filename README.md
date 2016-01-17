# virtualbox_usb_mitm

For some resone, I want to create a USB Man-In-The-Middle function. 
My idea is :

[Real USB Device]  <---> [USB-MITM] <---> [Host OS]

But I found this is not easy to achieve this purpose.
- For Windows, there used to be a Device Simulation Framework on Windows 7. https://msdn.microsoft.com/en-us/library/ff538304(v=vs.85).aspx As a part of Windows Driver Kit, it can simulate all kinds of USB device. The implementation could be:
[Window DSF generate USB traffics] <---> [Host] 
But this module was deprecated since Windows 8. 

- For Linux there is USB Gadget, which enable USB port works under device mode instead of host mode as it usually do. 
[USB Real Device] <----> [USB Host <---> USB Gadget] <---> [Final Host]
To use USB Gadget the USB controller should support USB OTG. For most desktop for laptop computers, the USB ports are always work under host mode, so it might not support USB OTG. And USB Gadget is usually not enable by default, user might need recompile the linux kernel to enable it.

Some Raspberry-Pi-like-PC can be configured to be a USB-MITM. BeagleBone support USB Gadget by default. There is a USBProxy project based on BeagleBone: https://github.com/dominicgs/USBProxy
The only drawback is ...$35 cost is kinds of high...



