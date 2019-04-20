# Eve-NG-in-Google-Cloud :cloud:
How to install Eve-NG in Google Cloud. 

![Google Cloud Dashboard](https://github.com/NetDevNotes/Eve-NG-in-Google-Cloud/blob/master/gcloud-dashboard.png)

#### Create a Googe Cloud account which will give you $300US free credit :smile:
https://cloud.google.com/

* Log in
* Click `Go To Console`
* Click `Select a project`
* Click `New Project`
* Project name = `eve-ng`
* Click '`Create`

> After a short wait, notice the project now says `eve-ng` 

> You are now back at the customisable dashboard for this project. 

Click `Activate Cloud Shell` (top right toolbar) 

> Below command creates the nested virtualization supported image based on `Ubuntu 16.04 LTS`. 

Paste the below into the cloud shell terminal: 

```gcloud compute images create nested-virt-ubuntu --source-image-project=ubuntu-os-cloud --source-image-family=ubuntu-1604-lts --licenses="https://www.google.com/compute/v1/projects/vm-options/global/licenses/enable-vmx”```

> The above command is invoking gcloud’s compute resources and creating an ubuntu image named nested-virt-ubuntu using the ubuntu-os-cloud source image with ubuntu version 16.4 which is what eve-ng requires.  It will not work on version 18 for example.  Then the last part is the license and where vmx (nested virtualisation) is activated. 

Say `yes` to the below message if you receive it: 

```
API [compute.googleapis.com] not enabled on project [838774168112].Would you like to enable and retry (this will take a few minutes)?(y/N)?
```

You will see the below output:

```
Enabling service [compute.googleapis.com] on project [838774168112] Waiting for async operation operations/acf.4c7e21aa-6f18-4286-b0f4-10fba7115f88 to complete
Operation finished successfully. The following command can describe the Operation details: gcloud services operations describe operations/tmo-acf.4c7e21aa-6f18-4286-b0f4-10fba7115f88Created [https://www.googleapis.com/compute/v1/projects/eve-ng-238108/global/images/nested-virt-ubuntu].NAME PROJECT FAMILY DEPRECATED STATUSnested-virt-ubuntu eve-ng-238108 READY
```

> The `READY` status indicates a successful install

* Click the 3 bars on the top left and select `Compute Engine` 
* Click `Create`

Enter the below properties. 

Name = `eve-ng` 
Region = `us-east1 (South Carolina)`
Zone = `us-east1-b`

> You can choose other Regions, but some are premium tier and more costly for the networking. South Carolina is a standard tier.  This is the same for Zones. Moving between Regions is possible, but is not just a case of selecting a new region.  An export is required.  The costs are shown on the right. Australia is more costly, but of course choosing a region closer to you is better from a latency perspective. 

> South Carolina currently has a $24.67 monthly estimate, whereas Australia is approximately $34.98. 

> The more vCPU’s you use will increase the monthly cost. But when the instance is off you will not be charged.  Note that there are some services which even when the instance is off generates charges, static IPs and persistent storage for example. But the bulk of fees are from compute uptime. 

> To run IOS-XR, IOS-XRV or CSR1000v or a lot of NXOS devices increase the CPU’s and memory. The below customised settings are an example. 

* 'Machine Type' Leave it at ‘1 vCPU’ and click 'Customise'
* With the sliders select `4 vCPU` and `4GB of Memory`. 
* CPU platform = `Intel Skylake or later` 
* Boot disk = Click `Change` and select the images we created in the previous step. 
* Click `Custom Images` & select `eve-ng` 
* Click radio button `nested-virt-ubuntu` 
* Boot disk = `Standard persistent disk`
* Size (GB) = This depends on you and how many eve-ng images and projects you will be using. Enter `25GB` 
* Firewall = `Allow HTTP traffic` 
* Click on `Management, security, disks, networking, sole tenancy` 
* Click `Networking`
* Click the `pencil`
> We will use and internal and an external IP 
> Selecting a static external IP is more convenient, but has additional costs.  Select the default `Ephemeral` 
* Primary internal IP = `Ephemeral (Automatic)` 
* External IP = `Ephemeral` 
* Network Service Tier = `Standard`
> Because we chose 'South Carolina' we get the option to choose the cheaper `standard tier` 
> Premium provides better HA, better failover, AnyCast IPs and more features. Standard is more a best effort service. 
* Click `Done`
* Click `Create`
> This will create and start the VM. 
> After a while you will see the VM’s details including its Internal and External IP. 
> The 3 vertical disclosure dots on the right will give you options including `Start`, `Stop` and `Delete`, amongst others. 
> You can also download the `Google Cloud Console app` to access your VM’s from your phone to start and stop them. Might come in handy when you forget you left an instance running with 16 vCPU’s and 100GB of memory.
> Create firewall rule to allow `tcp:32000-65535` through to your VPC
* Navigate to: `Google Cloud Console > VPC Network > Firewall rules`
* Create the below rule:

Name | Type | Description | Filters | Protocols/Ports | Action | Priority | Network
---- | ---- | ----------- | -------  | ---------------  | ------  | -------- | ------- 
eve-ng-inbound | Ingress | Allow ports for eve-ng | home-ip-address | tcp:32000-65535 | Allow | 1000 | default

* Navigate back to `Compute Engine > VM Instances`
* Click on `SSH` 
> There will be a key exchange and you will be connected. 
* Assume root and set a password: 

```
nico@eve-ng:~$ sudo -s 
root@eve-ng:~# passwd 
Enter new UNIX password:  
Retype new UNIX password:  
password updated successfully
```
* Allow root to log in via ssh 
`root@eve-ng:~# vim /etc/ssh/sshd_config`
* Change `PermitRootLogin` to `yes` 
```
Authentication:
PermitRootLogin yes
```
* Change below to yes to allow password authentication

`PasswordAuthentication yes`

> Using keys is a better option than password auth.

* Restart ssh 
`service sshd restart` 

* Now should be able to log in via ssh your terminal emulator.  
* Next eve-ng requires the first NIC to be named `eth0` 
* Note it is currently called `ens4`

```
root@eve-ng:~# ifconfig 
ens4      Link encap:Ethernet  HWaddr 42:01:0a:8e:00:02   
          inet addr:10.142.0.2  Bcast:10.142.0.2  Mask:255.255.255.255 
          inet6 addr: fe80::4001:aff:fe8e:2/64 Scope:Link 
          UP BROADCAST RUNNING MULTICAST  MTU:1460  Metric:1 
          RX packets:2418 errors:0 dropped:0 overruns:0 frame:0 
          TX packets:1899 errors:0 dropped:0 overruns:0 carrier:0 
          collisions:0 txqueuelen:1000 
          RX bytes:862718 (862.7 KB)  TX bytes:230363 (230.3 KB) 
```
* Rename the NIC
```
root@eve-ng:~# vim /etc/udev/rules.d/70-persistent-net.rules
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="42:01:0a:8e:00:02", NAME=“eth0" 
```
* Reboot 
`root@eve-ng:~# shutdown -r now`

> Next download the gpg.key, install the new repository and then install eve-ng. Run the below commands individually, and when installing eve-ng answer be conscious it will ask you for a new MySQL password, just accept the default and press enter.

```
root@eve-ng:~# wget http://www.eve-ng.net/repo/eczema@ecze.com.gpg.key 
root@eve-ng:~# apt-key add eczema@ecze.com.gpg.key 
root@eve-ng:~# apt update 
root@eve-ng:~# add-apt-repository "deb [arch=amd64] http://www.eve-ng.net/repo xenial main" 
root@eve-ng:~# apt update 
root@eve-ng:~# apt-get install eve-ng 
```

* Run the `apt-get install eve-ng` command again, you will notice more updates being installed. 
* When asked `What do you want to do about modified configuration file kernel-img.conf?` Select `keep the local version currently installed` 

`root@eve-ng:~# apt-get install eve-ng`

* Run `apt-get install eve-ng` one last time just to confirm all updates are now ‘Done’

```
root@eve-ng:~# apt-get install eve-ng 
Reading package lists... Done 
Building dependency tree        
Reading state information... Done 
eve-ng is already the newest version (2.0.3-95). 
0 upgraded, 0 newly installed, 0 to remove and 12 not upgraded. 
```
* Log out and back in as root 
* You will be presented with the Eve-NG Setup utility, click Control-C to break out to bash:

```
+-----------Root Password--------------+ 
| Type the Root Password:              | 
| +----------------------------------+ | 
| |                                  | | 
| +----------------------------------+ | 
+--------------------------------------+ 
|               <  OK  >               | 
+--------------------------------------+ 
```

* Open `Google Cloud Platform Dashboard` and naviage to `Compute Engine > VM Instances`, and click the `arrow` in the right corner of the external IP address, this will HTTP to eve-ng. 

* You should now be able to log on with the default username `admin` password `eve` 

> eve-ng requires the kernel to be 4.9.  If you say on the newer current kenrel and dont downgrade UKSM wont start. 

* Check what kernel you are using: 

```
root@eve-ng:~# uname -r 
4.15.0-1029-gcp
```
* Change the kernel by executing below commands, there might be a better way to do this, `let me know if so`: 

```
root@eve-ng:~# cd /boot/
root@eve-ng:/boot# ls -lah 
total 74M 
drwxr-xr-x  3 root root 4.0K Apr 19 11:27 . 
drwxr-xr-x 24 root root 4.0K Apr 19 11:32 .. 
-rw-r--r--  1 root root 202K Mar 22 14:36 config-4.15.0-1029-gcp 
-rw-r--r--  1 root root 197K Sep 14  2017 config-4.9.40-eve-ng-ukms-2+ 
drwxr-xr-x  5 root root 4.0K Apr 19 11:27 grub 
-rw-r--r--  1 root root  21M Apr 19 11:27 initrd.img-4.15.0-1029-gcp 
-rw-r--r--  1 root root  31M Apr 19 11:27 initrd.img-4.9.40-eve-ng-ukms-2+ 
-rw-------  1 root root 3.9M Mar 22 14:36 System.map-4.15.0-1029-gcp 
-rw-------  1 root root 3.5M Sep 15  2017 System.map-4.9.40-eve-ng-ukms-2+ 
-rw-------  1 root root 7.8M Mar 25 11:23 vmlinuz-4.15.0-1029-gcp 
-rw-------  1 root root 7.0M Sep 15  2017 vmlinuz-4.9.40-eve-ng-ukms-2+ 

root@eve-ng:/boot# mkdir ./old/
root@eve-ng:/boot# mv *4.15* ./old/
root@eve-ng:/boot# ls -lah 
total 41M 
drwxr-xr-x  4 root root 4.0K Apr 19 11:45 . 
drwxr-xr-x 24 root root 4.0K Apr 19 11:32 .. 
-rw-r--r--  1 root root 197K Sep 14  2017 config-4.9.40-eve-ng-ukms-2+ 
drwxr-xr-x  5 root root 4.0K Apr 19 11:27 grub 
-rw-r--r--  1 root root  31M Apr 19 11:27 initrd.img-4.9.40-eve-ng-ukms-2+ 
drwxr-xr-x  2 root root 4.0K Apr 19 11:45 old 
-rw-------  1 root root 3.5M Sep 15  2017 System.map-4.9.40-eve-ng-ukms-2+ 
-rw-------  1 root root 7.0M Sep 15  2017 vmlinuz-4.9.40-eve-ng-ukms-2+ 
```
> Notice all the 4.15 files have been moved. 

* Edit grub to keep the interface naming and also add `noquiet` 

```
root@eve-ng:/boot# sed -i -e  's/GRUB_CMDLINE_LINUX_DEFAULT=.*/GRUB_CMDLINE_LINUX_DEFAULT="net.ifnames=0 noquiet"/' /etc/default/grub 
```
* Update grub, notice it found the `4.9` kernel version
```root@eve-ng:/boot# update-grub 
Generating grub configuration file ... 
Found linux image: /boot/vmlinuz-4.9.40-eve-ng-ukms-2+ 
Found initrd image: /boot/initrd.img-4.9.40-eve-ng-ukms-2+ 
done 
```
* Restart 
`root@eve-ng:/boot# shutdown -r now`

> Notice 'System > System Status' page will now show UKSM as green. 

* SSH back into the instance (optional)

* Create a new non-root user, ssh to eve-ng (optional)
```root@eve-ng:~# sudo adduser nico
root@eve-ng:~# sudo usermod -a -G sudo nico
```
* Disable root from sshing (optional)
```root@eve-ng:~# vim /etc/ssh/sshd_config 
PermitRootLogin no 
```

* Copy a .qcow file to `/opt/unetlab/addons/qemu`. Any image copied here will show up in the eve-ng GUI, and you need to name each folder what holds the qemu file in a specific way listed [HERE](https://www.eve-ng.net/documentation/images-table)

> The start of the folder name is very important, and you need to include the trailing hyphen, the remainder of the folder name can be anything, but usually you would add the software version. So for a Nexus 9k the folder that houses the .qcow file must begin with `nxosv9k-` the final folder name in this example will be `nxosv9k-7.0.3.I7.2`

```
root@eve-ng:/opt/unetlab/addons/qemu# ls 
nxosv9k-7.0.3.I7.2
```

* SCP your qemu/qcow image to the appropriately name folder to `/opt/unetlab/addons/qemu`

> In my case the full path is below: 

`/opt/unetlab/addons/qemu/nxosv9k-7.0.3.I7.2/` 

* After you copy files its recommended to fix the permissions using the below command: 
`/opt/unetlab/wrappers/unl_wrapper -a fixpermissions`

* Browse to the web GUI 
[http://35.211.142.236](http://35.211.142.236)

* From File manager click on `Add new lab` and enter some details, then click `Save` 
* From the new lab, right click on a blank space and select `Node` to add a device.
* Select the `Cisco NX-OSv 9k` image
> From here you can change the nodes settings, hostname, CPU's etc.
* Click `Save`
* Right click the switch and click `Start`
* Double click the switch to connect to it.
> Notice when using SecureCRT for example, the default is a telnet connection on a high tcp port like port 32769.  This is what the previous firewall rule was for.

![eve-ng](https://github.com/NetDevNotes/Eve-NG-in-Google-Cloud/blob/master/system_status.png)

**EOF**
