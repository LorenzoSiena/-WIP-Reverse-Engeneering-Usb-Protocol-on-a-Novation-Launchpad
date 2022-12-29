# -WIP-Reverse-Engineering-Usb-Protocol-on-a-Novation-Launchpad
This is a proof of work on how to use and control a usb (1.0) device (Novation Launchpad) without using a prebuild driver.
# DISCLAIMER 
This software is provided 'as-is', without any express or implied warranty. In no event will the authors be held liable for any damages arising from the use of this software. The software is released under the GNU General Public License version 3.0 (GPLv3) and is provided without warranty. Use at your own risk."

# WHY
It has always seemed strange to me having to use a driver to use a device and having to submit to the benevolence of the manufacturer to be able to use it, and therefore having a LaunchPad aside I thought it would be useful to save it from obsolescence and be able to use it as I pleased.
Or at least being able to turn the lights on that.

# WHAT 
-Launchpad Mk1 (but could be any kind of usb device till 3.0(?))
-usb A 1.0 cable
-A Linux PC
-Python3
-Pyusb (or any other implementation of the usb protocol, this is just the simpliest and laziest way to send or receive messages)

# WHERE
The development environment will be linux Fedora but any other linux distro is fine.
Our intention is not to create an efficient c driver module to add to the kernel but just to send and receive messages.

# SETUP
After having installed all the dependencies ie python3 and pyusb, and having set our environment well,
we will have to write a script able to: set, find the end points,
send or receive, close the connection.

# THINKING
How does the usb protocol work?
Having known the other much simpler protocols such as UART, SPI etc... I was expecting something very similar...
Apparently not.

# ARCHITECTURE
"If you know yourself and your enemy, your victory is sure. If you know yourself but not your enemy, your odds of winning and losing are equal."
Sun Tzu

The usb protocol (in its basic and slow form in this case usb 1.0 but backwards compatible up to usb3(?)) can be briefly summarized in a Client (USB DEVICE) and Server (our PC) architecture.
Each request from the Server must be satisfied by the client who reacts with an IN or OUT message at a certain port (ENDPOINT) of the device.

Our first problem is finding the endpoint addresses (IN/OUT), which is easily solved with lsusb command:

```bash
$ ls
$ Bus 002 Device 008: ID XXXX:XXXX Focusrite-Novation Launchpad
```

"Bus" indicates the USB port that the device faces.
"Device" the number of the device mounted on the system.
Using these two arguments  

```bash 
lsusb -D /dev/bus/usb/002/080
```
we will get an answer:

 #### WARNING!
do you see this arrow? -> 
it was added under study and should be read as a comment

 #### WARNING pt2!
There is a lot to read but we are only interested in a FEW highlighted lines.

```bash
Device: ID 1235:000e Focusrite-Novation Launchpad
Couldn t open device, some information will be missing

Device Descriptor:
  bLength                18				-> Size of the Descriptor in Bytes (18 bytes)
  bDescriptorType         1				-> Device Descriptor (0x01)
  bcdUSB               1.00				-> USB Specification Number which device complies too.MAX USB VERSION SUPPORTED [USB 2.0 is reported as 0x0200, USB 1.1 as 0x0110 and USB 1.0 as 0x0100.]
  
  [these 3 are used by the operating system to find a class driver for your device.]
  
  bDeviceClass          255 Vendor Specific Class 	->Class Code (Assigned by USB Org) If equal to 0xFF, the class code is vendor specified. [LOCKED ON HARDWARE]
  bDeviceSubClass         0  				->Subclass Code (Assigned by USB Org)
  bDeviceProtocol       255 				->Protocol Code (Assigned by USB Org)
  
  bMaxPacketSize0         8				->Maximum Packet Size for Zero Endpoint. Valid Sizes are 8, 16, 32, 64
  idVendor           0x1235 Focusrite-Novation 		->Vendor ID (Assigned by USB Org) [used by the operating system to find a driver for your device]
  idProduct          0x000e Launchpad	       		->Product ID (Assigned by Manufacturer) [used by the operating system to find a driver for your device]
  bcdDevice            0.02				->Device Release Number [device version number.This value is assigned by the developer]
  iManufacturer           1 Novation DMS Ltd		->Index of Manufacturer String Descriptor [details Manufacturer]
  iProduct                2 Launchpad			->Index of Product String Descriptor	  [details Product]
  iSerial                 0 				->Index of Serial Number String Descriptor [details Serial]
  bNumConfigurations      1				->Number of Possible Configurations
  
  
  
  Configuration Descriptor:     ->[it is possible to have two configurations, one for when the device is bus powered and another when it is mains powered.]
    bLength                 9 				->Size of Descriptor in Bytes
    bDescriptorType         2				->Configuration Descriptor (0x02)
    wTotalLength       0x0020				->Total length in bytes of data returned (32BYTE) [The wTotalLength field reflects the number of bytes in the hierarchy]
    bNumInterfaces          1				->Number of Interfaces [specifies the number of interfaces present for this configuration]
    bConfigurationValue     1				->Value to use as an argument to select this configuration [is used by the SetConfiguration request to select this configuration]
    iConfiguration          0 				->Index of String Descriptor describing this configuration [is a index to a string descriptor describing the configuration in human readable form]
    bmAttributes         0x80          			-> 128 [specify power parameters for the configuration]
      (Bus Powered)
    MaxPower              430mA        -> maximum consumption Configuration
    
    	Interface Descriptor:                   ->A GROUP of ENDPOINTS that together process a DEVICE FUNCTION
    	  bLength                 9			->Size of Descriptor in Bytes (9 Bytes)
    	  bDescriptorType         4			->Interface Descriptor (0x04)
    	  bInterfaceNumber        0			->Number of Interface [This should be zero based, and incremented once for each new interface descriptor.]
    	  bAlternateSetting       0			->Value used to select alternative setting
    	  bNumEndpoints           2			->Number of ENDPOINTS used for this interface [This value should exclude endpoint zero and is used to indicate the number of endpoint descriptors to follow][1+2]
    	  bInterfaceClass       255 Vendor Specific Class ->Class Code (Assigned by USB Org)
    	  bInterfaceSubClass      0 			->Subclass Code (Assigned by USB Org)
    	  bInterfaceProtocol      0 			->Protocol Code (Assigned by USB Org)
    	  iInterface              0 			->Index of String Descriptor Describing this interface [string description of the interface]
    	  	
    	  	[Endpoint descriptors are used to describe endpoints other tha Endpoint zero which is ALWAYS assumed to be a control endpoint and is configured before any descriptors are even requested.]
    	  	
    	  	Endpoint Descriptor:
    	    	bLength                 7 		->Size of Descriptor in Bytes (7 bytes)
    	   	 bDescriptorType         5		->Endpoint Descriptor (0x05)
    	    	bEndpointAddress     0x81  EP 1 IN	->ADDRESS 129 (0x81) ACCEPT INPUT VALUES 
    	    	bmAttributes            3
    	     	 Transfer Type            Interrupt	-> type INTERRUPT
    	     	 Synch Type               None 		->NO SYNCH
    	     	 Usage Type               Data		->ACCEPT INPUT VALUES(00) [00 = Data Endpoint,01 = Feedback Endpoint,10 = Explicit Feedback Data Endpoint,11 = Reserved]
    	   	 wMaxPacketSize     0x0008  1x 8 bytes	->Maximum Packet Size this endpoint is capable of sending or receiving (8 byte=8 file da 8 bit=64bit)
    	    	bInterval              10		->Interval for polling endpoint data transfers 10ms
```
The path we have to travel to reach an endpoint and finally be able to communicate is quite straight forward and will be :
DeviceDescriptor->ConfigurationDescriptor->InterfaceDescriptor->Endpoint
where
DeviceDescriptor= Our Device
ConfigurationDescriptor= The only one we have 



# PROTOCOLS

# HACKING

# NiceTrick

     ## A LCD SCREEN?
     ## TESTRIS?
     ## MUSIC!
