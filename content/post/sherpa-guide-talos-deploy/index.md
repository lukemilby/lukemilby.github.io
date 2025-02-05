---
title: "Sherpas Guide to Talos: Deploy"
date: 2025-02-05
draft: false
tags:
  - Talos
  - Talosctl
  - Netboot
  - k8s
---


[Talos](https://www.talos.dev/) is an operating system that runs k8s and thats all it does. It's meant to lower the attack suface on the node it's running. The OS is immutable and the size is mini.

In this part of the journey we'll walk through long term deployment strategy for Talos with PXE. 
PXE Boot as I've heard it called, sends a boot menu over to the node through the network. Where you select what image to run. Please note you need a network card that support PXE boot. 

In my last post [PXE Booting Talos](https://www.lukeslab.io/post/pxe-talos/) I talked about setting up a PXE boot server. This one we setup a new image using the Image factory and automate the deployment of Talos with just a few lines to update. 


### Image Factory

Talos being even more odd has this [Image Factory](https://factory.Talos.dev/) service to build Talos images. Odd but fucking cool! Its a god damn barkery for Talso Images. So the main reason you'd want to come bake an image is [Extensions](https://www.Talos.dev/v1.9/Talos-guides/configuration/system-extensions/), oh and updates. Extenstions extend the Talos root filesystem. Giving you access to thinks like ISCSI, Nvidia Drivers and so on. [Extensions Catalog](https://github.com/siderolabs/extensions?tab=readme-ov-file#extension-catalog)

You'll still see references to extension configuration in the documentation but that's no longer used. You have to build them into the image.

### Missing iscsiadm

So for me I was using the base image but wanted to use Longhorn for my storage service. I was reading that that amount of node I'd have to set ceph/rook would be a waste. Unknowingly I installed Longhorn and was met with logs from the Longhorn pods crying. When Longhorn is deployed its looking for access to the nodes harddrive as well as `iscsiadm`. iscsiadm is a utility used to discover and access iSCSI devices. Longhorn uses those harddrives to create Volumes, when a persitant volume claim is made. Those Volumes are distributed throught the cluster where data is replicated. 

So we know the extenstion exists for Talos from looking over the Extensions Catalog. [iscsi-tools extenstion](https://github.com/siderolabs/extensions/tree/main/storage/iscsi-tools). 

### Extensions
You could find some documentation on extension that will talk about enabling them in the config. I tried deploying with the config

``` 
Talosctl apply-config -n 10.10.20.226 --file ../Talos/worker.yaml 
WARNING: .machine.install.extensions is deprecated please see https://www.Talos.dev/latest/Talos-guides/install/boot-assets/
```

So whats next? The Image Factory.

I was deploying Talos already. I had a single control node and one worker during this time. Images where deploy with Netboot.xyz and it was manual process. You'd boot naviage the menu find Talos and confirm. Not bad but I had 3 more nodes waiting to be setup and deployed. 

Next step was to remove the human interaction. I needed to get automated deployments with PXE. Before we can automate tho we should get an image from the Factory.

Image Factory can build in extensions. Its a pretty straight forward process.

Hardware -> Version -> Arch -> Extensions -> Customization -> Link to Talos Image


When you get to the last page with the Image Links its not just an ISO. You get raw and qcow2 along with.... and iPXE script!

Go to that link
https://pxe.factory.Talos.dev/pxe/376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba/v1.9.3/metal-amd64


### Talos iPXE script
```bash
#!ipxe

imgfree
kernel https://pxe.factory.Talos.dev/image/376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba/v1.9.3/kernel-amd64 Talos.platform=metal console=tty0 init_on_alloc=1 slab_nomerge pti=on consoleblank=0 nvme_core.io_timeout=4294967295 printk.devkmsg=on ima_template=ima-ng ima_appraise=fix ima_hash=sha512
initrd https://pxe.factory.Talos.dev/image/376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba/v1.9.3/initramfs-amd64.xz
boot
```

This tells your computer to download the kernel and initramfs then boot from it.
initramfs is the first root filesystem your machine has to access. Feel free to explore that more on your own time.

I only know enough to get this working. Before the kernel and initramfs are downloaded, we to clean the chained ipxe operations. To do so we use `imgfree`. imgfree well... frees up resources so you can boot.

I found this out while trying to upgrade Talos and using the Image Factory.The generated config didnt contain that. So when I was setting up the menu it would download the needed files and break because it ran out of memory. 

Whats going on is the image downloads but doesn't auto free from the network card so the next time it would boot and pull down those files it would run out of space.

I ended up make a commit in Talos image factory. Closed the issue once someone brought up this might not be the fix for it. I thought okay... I dont understand ipxe enough, but it got revied and the maintainer merged it. 

https://github.com/siderolabs/image-factory/pull/191


### Netboot

We have our ipxe config. Lets go boot from it. 

Netboot was the iPXE server I have running in [docker](https://netboot.xyz/docs/docker). Navigate to the web interface of Netboot. Default port is `3000`.

We'er going to modify 2 files already in Netboots menus and create a new one for our Talos ipxe file.


#### Configure boot.cfg

```bash
# set default boot timeout
set boot_timeout 100000
```
When you're booting into a menu you have about 300 seconds to pick an a menu item by default. I dropped this down to 100 seconds.


#### Create Talos IPXE File

Next is to create the Talos ipxe config.

Select `Menus` again.

Drop in a name, `Talos-image.ipxe` and `Create New`

Copy and Paste exactly what Talos Image Factory gave you.


#### Update menu.ipxe

Last it's time to wedge ourselfes in this here `menu.ipxe` file. This is the default menu from Netboot.

<img src="/Talos/ipxe-menu.png" alt="iPXE Menu">

The only thing we need to do is change


```
item --gap Default:
item local ${space} Boot from local hdd
item --gap Distributions:
```

```
item --gap Default:
item Talos-image ${space} Boot to Talos
item --gap Distributions:
```

It's was a lot more confusing in my head. Replacing local -> Talos-image. 

ipxe reminds me a lot of qbasic...

Now we still have options from the menu but also default boot into the Talos ipxe script. 

You can find all the commands for IPXE here https://ipxe.org/cmd

### Deployment

We're ready to deploy. I find its best to test my ipxe script with a VM over baremetal. Just slide into proxmox and create a VM. I used a debian image and keep all the defaults. As soon as it boots its starting to boot from Netboot. This gives you a simple way to test. 

Once your through testing the menus hit that `ctrl+alt+delete` and reset the session till you ge what you want.

Next: **Talos Configuration**

### TL:DR

1. Generate Image from the Facotry: https://factory.talos.dev/

   a. Add Extensions as needed

2. Copy the contents from the iPXE script it generates to your Netboot.xyz instance
3. Update Netboot Menus
   a. Update boot.cfg
      ```bash
      # set default boot timeout
      set boot_timeout 100000
      ```
   b. Update menu.ipxe

      **Before**
      ```bash
        item --gap Default:
        item local ${space} Boot from local hdd
        item --gap Distributions:
      ```

      **After**
      ```
        item --gap Default:
        item Talos-image ${space} Boot to Talos
        item --gap Distributions:
      ```
4. Test and ensure your menu setup is working as expected with a VM.