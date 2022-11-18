# Part 1: Satellite 6.12 Installation   

[Tutorial Menu](https://github.com/pslucas0212/RedHat-Satellite-6.12-VM-Provisioning-to-vSphere-Tutorial/blob/main/README.md)

Red Hat Satellite is a powerful content management and provisioning tool that you can add to any Red Hat Enterprise Linux subscription with the addition of a Smart Management subscription.  With Red Hat Satellite you can curate specific content across mutliple lifecycle environments through out your entire RHEL enviroment whether it is on-prem, in the cloud or hybrid.  In fact you can use Red Hat Satellite with your market-place instances of RHEL.  

In this multi-part tutorial we will cover how to provision RHEL VMs to a vSphere environment from Red Hat Satellite 6.12.  We will focus on provisioning RHEL 8.7 and 9.1 VMs using two Satellite RHEL life cycle envrionments, and you can easily adapt what you learn here to provision other RHEL versions.

In part 1, I'm documenting the steps for a simple "lab" install of Satellite 6.12.  The purpose of this setup is to give you a quick hands-on experience with Satellite.  The lab infrastructure is deployed to a small vSphere 7.0 lab environment with three EXSi servers that have internet access for the installation.  For this lab, I setup a separate server to host DNS and DHCP services for the network that is hosting Satellite and the vSphere environment.  Also, in a production environment you would also want to configure Satellite to interact with your directory/security services.  

I would recommend creating a local time server and configure all system in this lab environment to use the same local time source.


### Pre-Reqs


Create a VM for Satellite and install RHEL 8.7.  The VM was sized with 4 vCPUS, 20GB RAM and 400GB "local" drive.  Note: For this example I have enabled Simple Content Access (SCA) on the Red Hat Customer portal and do not need to attach a subscription to the RHEL or Satellite repositories.  After you have created and started the RHEL 8.7 VM, we will ssh to the RHEL VM and work from the command line.

For this lab environment I chose sat01.example.com for the hostname of the server hosting Satellite. 

Check hostname and local DNS resolution.  Use dig to test forward and reverse lookup of the server hosting Satellite.  If the Satellite hostname is not available from DNS, the initial installation will fail.    
```
# ping -c3 localhost
# ping -c3 sat01.example.com -f
# dig sat01.example.com +short
# dig -x 10.1.10.254 +short
```   
We will set the hostname on the server to avoid any future issues.
```
# hostnamectl set-hostname sat01.example.com
```

Verify the time server with chrony.  I have a local time server that my systems use for synching time.  Type the following command to check the the time synch status.  
```
# chronyc sources -v
```
Register Satellite Server to Red Hat Subscription Management service.
```
# sudo subscription-manager register --org=<org id> --activationkey=<activation key>
```
You can verify the registration with the following command.
```
# sudo subscription-manager status
```    
#### Configure and enable repositories  

With SCA, we still need to enable relevant repositories for our RHEL instances.  Following steps will walk you through enabling repos.

Disable all repos.
```    
# sudo subscription-manager repos --disable "*"
```       
Enable the following repositories.
```    
# subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rpms \
--enable=rhel-8-for-x86_64-appstream-rpms \
--enable=satellite-6.12-for-rhel-8-x86_64-rpms \
--enable=satellite-maintenance-6.12-for-rhel-8-x86_64-rpms
```
Verify that repositories are enabled.
```
# sudo subscription-manager repos --list-enabled
```
Enable the Satellite module.
```
# dnf module enable satellite:el8
```

Update all packages.  This may take a few minutes to complete.
```
# dnf update
```
Install SOS package on base OS for initial systems analysis in case you need to collect problem determination for any system related issues.  
```
# dnf install sos
```
 I would also recommend registering this server to Insights.  
```
# insights-client --register
```

Update the firewall rules for Satellite.
```
# firewall-cmd \
--add-port="53/udp" --add-port="53/tcp" \
--add-port="67/udp" \
--add-port="69/udp" \
--add-port="80/tcp" --add-port="443/tcp" \
--add-port="5647/tcp" \
--add-port="8000/tcp" --add-port="9090/tcp" \
--add-port="8140/tcp"
```

Make the firewall changes permanent
```
# sudo firewall-cmd --runtime-to-permanent
```

Verify the firewall changes
```
# sudo firewall-cmd --list-all
```

### Satellite Installation
Install Satellite Server packages and then install Satellite.  
```     
# dnf install satellite
```

We will run the initial satellite-installer to create a userid and password.  Later in this tutorial we will configure Satellite to use external DNS and DHPC services.  This initial configuration will take several minutes to complete.  
```
# satellite-installer --scenario satellite \
--foreman-initial-organization "Operations Department" \
--foreman-initial-location "Moline" \
--foreman-initial-admin-username admin \
--foreman-initial-admin-password Passw0rd! \
--foreman-proxy-dhcp-managed false \
--foreman-proxy-dns-managed false
```
If the installation is progressing successfully, your screen output will look similar to the following example.
```
2022-11-18 16:10:17 [NOTICE] [root] Loading installer configuration. This will take some time.
2022-11-18 16:10:20 [NOTICE] [root] Running installer with log based terminal output at level NOTICE.
2022-11-18 16:10:20 [NOTICE] [root] Use -l to set the terminal output log level to ERROR, WARN, NOTICE, INFO, or DEBUG. See --full-help for definitions.
2022-11-18 16:13:07 [NOTICE] [configure] Starting system configuration.
2022-11-18 16:13:42 [NOTICE] [configure] 250 configuration steps out of 1530 steps complete.
 2022-11-18 16:14:18 [NOTICE] [configure] 500 configuration steps out of 1531 steps complete.
2022-11-18 16:14:21 [NOTICE] [configure] 750 configuration steps out of 1535 steps complete.
2022-11-18 16:15:33 [NOTICE] [configure] 1000 configuration steps out of 1560 steps complete.
2022-11-18 16:16:02 [NOTICE] [configure] 1250 configuration steps out of 1560 steps complete.
2022-11-18 16:22:59 [NOTICE] [configure] 1500 configuration steps out of 1560 steps complete.
2022-11-18 16:23:14 [NOTICE] [configure] System configuration has finished.
  Success!
  * Satellite is running at https://sat01.example.com
      Initial credentials are admin / Passw0rd!

  * To install an additional Capsule on separate machine continue by running:

      capsule-certs-generate --foreman-proxy-fqdn "$CAPSULE" --certs-tar "/root/$CAPSULE-certs.tar"
  * Capsule is running at https://sat01.example.com:9090

  The full log is at /var/log/foreman-installer/satellite.log
Package versions are being locked.
```

Use the following command to find the name of the Satellite server you just updated.
```
# hammer proxy list
```

See which services are configured on your Satellite server.  
```
# hammer proxy info --name sat01.example.com
```

### Login into the Satellite console  

We can now launch and login to the Satellite console by entering [http://sat01.example.com](http://sat01.example.com) for the Satellite url.  Satellite will redirect the browser to Satellite's secure login page.  You will need to accept Satellite's certificate for your browser.  

For this example we are using a local login.  For production work you will want to integrate your directory service with Satelllite. Enter the user id and password and click Login button.  

![Click Login button](/images/sat01.png)  

You are now at the Satellite home screen.  

![Satellite Home Srceen](/images/sat02.png)  



## References  
[Installing Satellite Server from a Connected Network](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.12/html/installing_satellite_server_from_a_connected_network/index)   
[Simple Content Access](https://access.redhat.com/articles/simple-content-access)  
[Provisioning VMWare using userdata via Satellite 6.3-6.6](https://access.redhat.com/blogs/1169563/posts/3640721)  
[Understanding Red Hat Content Delivery Network Repositories and their usage with Satellite 6](https://access.redhat.com/articles/1586183)

