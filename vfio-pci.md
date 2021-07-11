***The goal***
Connect a SCSI-2 device to a modern PC and pass through the SCSI controller to a virtual machine. Automated test scripts will be used to test the firmware of the SCSI-2 device with different operating systems (xxxBSD, Linux, MacOS X, etc). 

I haven't really found any SCSI-2 PCIe cards (because... why would any sane person want one?). So, I went with a PCIe to PCI bridge and an old SCSI controller.

***My setup:***

* Dell XPS 8940 (11th gen i7, 16GB RAM, 512GB NVMe, 8TB HDD)
* Ubuntu 21.04 (I'd typically use the latest LTS release, but v20 didn't work with the new 11th gen i7)
*  [Sintech PCI-E Express X1 to Dual PCI Riser Extender Card](https://smile.amazon.com/dp/B00KZHDSLQ/ref=cm_sw_r_tw_dp_BXXMCM06J2GQNWMGA6HP?_encoding=UTF8&psc=1)
*  [Adaptec PCI SCSI Controller Card AHA-2930CU](https://storage.microsemi.com/en-us/support/scsi/2930/aha-2930cu/)
* [RaSCSI SCSI-2 device emulator](http://www.rascsi.com)

***PCI-E to PCI adapter***

I installed the PCIe board into the top-most PCIe slot. When booting, the OS would hang waiting for the journal service to start.

Bottom slot - PC couldn't detect the boot drive

Middle slot - Success!! Downside was that it was the only 16 lane PCIe slot. 

***SCSI Controller & RaSCSI***

This *seemed* easy. Plug and play! The emulated SCSI drive immediately showed up. I could mount the emulated hard drive. Life was good! Time to set up VFIO!

***Setting up VFIO***
I validated that the only items in the IOMMU group were the SCSI controller and the PCI bridge. (Group 14)

```
lspci -vnn | egrep 'group|^0'
....
02:00.0 PCI bridge [0604]: Pericom Semiconductor PI7C9X111SL PCIe-to-PCI Reversible Bridge [12d8:e111] (rev 02) (prog-if 00 [Normal decode])
        Flags: bus master, fast devsel, latency 0, IRQ 255, IOMMU group 14
03:04.0 SCSI storage controller [0100]: Adaptec AIC-7850T/7856T [AVA-2902/4/6 / AHA-2910] [9004:5078] (rev 03)
        Flags: medium devsel, IRQ 255, IOMMU group 14
....
```


I followed a couple good tutorials online to set up VFIO. High level of what I did:
* Enable all of the Virtualization options in the BIOS
* Updated the kernel cmdline (/etc/default/grub). Enabled intel_iommu and added the PCI IDs that VFIO should take control of
* Created file /etc/modprobe.d/vfio.conf which set the options to vfio_pci to set the PCI IDs. Also, set a soft dependency between the aic7xxx and scsi_transport_spi kernel modules and the vfio-pci module.
* Created file /etc/modules-load.d/vfio-pci.conf to tell the kernel to load the vfio-pci module.
* Updated /etc/modules to load vfio and vfio_pci *(Maybe this was a duplicate of the previous step?)*

To validate things, I ran `lspci -vnn` to verify that the "active kernel driver" was vfio-pci

```
03:04.0 SCSI storage controller [0100]: Adaptec AIC-7850T/7856T [AVA-2902/4/6 / AHA-2910] [9004:5078] (rev 03)
...
        Kernel driver in use: vfio-pci
        Kernel modules: aic7xxx
```

Life is good, right? Wrong....

I startup up virt-manager, configured a new virtual machine, then added the Adaptec SCSI card as a "PCI Host Device". Launch the virtual machine and.....
```
Error starting domain: internal error: qemu unexpectedly closed the monitor: 2021-07-10T23:13:06.946826Z qemu-system-x86_64: -device vfio-pci,host=0000:03:04.0,id=hostdev0,b
us=pci.5,addr=0x1: vfio 0000:03:04.0: Failed to set up TRIGGER eventfd signaling for interrupt INTX-0: VFIO_DEVICE_SET_IRQS failure: Device or resource busy
```

I installed a random PCI Ethernet card, repeated the configuration process, and the Ethernet card worked! The VM's OS detected the Ethernet card and everything was great. Try to add the SCSI PCI card in the VM configuration... same error.

I dug out another PCI SCSI card (Tekram DC-390), repeated the configuration process, and same result as the Adaptec one.

After lots of Googling, the best I could find were forum posts saying "get a new card".

But wait... would it matter which PCI slot in the bridge? In all of my testing, I only used the one slot since it was easier to get to. So, I moved it to the other PCI slot in the bridge card, re-configured the VM (since it is now device 03:05.0) and Bam! It works!

***Lesson of the day***
PCI is black magic. When messing with vfio and legacy hardware, try different combinations of slots/positions for the cards. It might save the day.

***Files that I updated/created***

* (/etc/default/grub)[files/vfio/default.grub]
* (/etc/modprobe.d/vfio.conf)[files/vfio/modprobe.d.vfio.conf]
* (/etc/modules-load.d/vfio-pci.conf)[files/vfio/vfio-pci.conf]
* (/etc/modules)[files/vfio/etc.modules]


