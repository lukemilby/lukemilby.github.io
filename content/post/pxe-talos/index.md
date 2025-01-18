---
title: "PXE booting Talos"
date: 2025-01-11T10:48:16-05:00
draft: false
tags:
  - PXE
  - Talos
  - Netboot
  - Docker
  - k8s
---

Goal: Setup Netboot to PXE Boot Talos and migrate my existing lab servcies to k8s

**Objectives**
- What is Talos
- Why PXE Boot
- How to setup Netboot in Docker
- Deploying Talos
- Closing

## What is Talos?

TLDR: Small (100mb) Operating System for Kubernetes

Makes it a one stop shop to deploy Kubernetes through live booting from an IOS. 

Talos live boots off an ISO and gets to working. Leaving your system untouched. So once you've overstayed your welcome minikube I'm sure you'll stumble upon Talos. If not, Talos IMO is the next step after minikube and k3s should be over looked. You can also use kubeadm, provision Ubuntu VM have k8s running on a VM which seems only good if you're testing.

Talos has me locked in for deploy it to my lab. To do this I'll be using Lenovo ThinkCentre M93p's. I have 2 right now but expanding to 5 for just 150$ more. How we deploy is through PXE Boot (Preboot eXectution Environment).

## Why PXE(pixie) Boot

It's for longevity. PXE is the alternative to using a USB. You'd have to swap your bootable USB for every node and do the same thing when the power gos out. Alternativly you could just make a bootable USB for each but lets not spend any money. It's also nice just having PXE ready fro VMs. 

PXE Flow

PC boot > DHCP > PXE Server > Boot instruction > Select Image

Machine boots, talks with DHCP about where the PXE server is, PXE boot pushes boot instruction over the network using TFTP. Your machine will take it from there. Downloading the right image and booting into an ISO. 

It makes even more sense for Talos. Its small, takes no time to deploy and makes adding a new bare metal server pain less. 

You can also remove the USB drives. 

## How to setup Netboot in Docker

The quickest way to get this up and running is deploying a docker container of [netboot.xyz docker setup](https://netboot.xyz/docs/docker)

```docker
docker run -d \  
	--name=netbootxyz \  
	-e MENU_VERSION=2.0.76 `# optional` \  
	-e NGINX_PORT=80 `# optional` \  
	-e WEB_APP_PORT=3000 `# optional` \  
	-p 3000:3000 `# sets web configuration interface port, destination should match ${WEB_APP_PORT} variable above.` \  
	-p 69:69/udp `# sets tftp port` \  
	-p 8080:80 `# optional, destination should match ${NGINX_PORT} variable above.` \  
	-v ./config:/config `# optional` \ 
	-v ./assets:/assets `# optional` \ 
	--restart unless-stopped \  
	ghcr.io/netbootxyz/netbootxyz
```


I'm a fan of having a composer file over just the straight docker command. Out of the box this docker run command is good to go. 

You can find that here [docker-compose.yaml](https://raw.githubusercontent.com/netbootxyz/docker-netbootxyz/refs/heads/master/docker-compose.yml.example)

Once your image has been deployed navigate to `http://<netboot container ip>:3000` 


<img src="/talos/udm-network-boot.png" alt="UDM Network Boot Setup">

## Deploying Talos from a network jack

DHCP will provide an IP and also tell the host where to find the boot file if the network card is ready to boot from PXE. 

My DHCP is managed by UDM Pro in my lab. 


<img src="/talos/netboot-ui.png" alt="testing">

1. Login to UDM Pro
2. Navigate to your Network Tab and find the network you want PXE setup on
3. Find DHCP Service Management
	1. Check Network Boot
	2. Assign your PXE Boot server and the Boot file it should load. You have 2 options to start before creating your own custom boot file
		1. netboot.xyz.efi
		2. netboot.xyz.kpxe

I ended up using **netboot.xyz.kpxe**, the .efi wasn't working as expected. I was running into issue with EFI. This is formats used for UEFI systems and while my boxes have UEFI and Legacy, setting it up for UEFI boot didn't seem to work. So I ended up using KPXE for legacy BIOS system


<img src="/talos/PXE-Talos.gif">

Heres an example of what booting from a PXE Server looks like. I power on the VM, DHCP gives me an IP and points me to the PXE server boot menu. I navigate to Talos and I'm up and running after the image was downloaded.


## Closing

I have 3 more nodes to add. **Lenovo ThinkCentre M93p Tiny i5-4570T 16GB 160GB SSD** from Ebay at 50$ a pop. Much cheaper than buying a Raspberry Pi. 2 are already in the lab with Talos booted in Maintinance mode. Next is getting Talosctl setup with those 2 nodes. 

Netboot out of the box is great not really much more I need to setup, but one glaring hole I need to fill is have PXE auto select Talos after a certain time limit. Right now I have to naviage a menu and find Talos, it would be easier if thats was the default image selected. Man to think Talos could replace all my proxmox servers...

