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

Update all packages.
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
# sudo yum install satellite
```

We will run the satellite-installer to create a userid and password along with the information to configure the DNS, DHCP and TFTP services.  This will take several minutes to complete.  
```
# satellite-installer --scenario satellite \
--foreman-initial-admin-username admin \
--foreman-initial-admin-password Passw0rd! \
--foreman-proxy-dhcp true \
--foreman-proxy-dhcp-managed true \
--foreman-proxy-dhcp-gateway "10.1.10.1" \
--foreman-proxy-dhcp-interface "ens192" \
--foreman-proxy-dhcp-nameservers "10.1.10.254" \
--foreman-proxy-dhcp-range "10.1.10.149 10.1.10.199" \
--foreman-proxy-dhcp-server "10.1.10.254" \
--foreman-proxy-dns true \
--foreman-proxy-dns-managed true \
--foreman-proxy-dns-forwarders "10.1.1.254" \
--foreman-proxy-dns-interface "ens192" \
--foreman-proxy-dns-reverse "10.1.10.in-addr.arpa" \
--foreman-proxy-dns-server "127.0.0.1" \
--foreman-proxy-dns-zone "example.com" \
--foreman-proxy-tftp true \
--foreman-proxy-tftp-managed true
```
If the installation is progressing successfully, your screen output will look similar to the following example.
```
2021-11-03 15:48:05 [NOTICE] [root] Loading default values from puppet modules...
2021-11-03 15:48:08 [NOTICE] [root] ... finished
2021-11-03 15:48:09 [NOTICE] [root] Running validation checks
2021-11-03 15:50:50 [NOTICE] [configure] Starting system configuration.
  The total number of configuration tasks may increase during the run.
  Observe logs or specify --verbose-log-level to see individual configuration tasks.
2021-11-03 15:51:01 [NOTICE] [configure] 100 out of 2460 done.
2021-11-03 15:51:01 [NOTICE] [configure] 200 out of 2460 done.
2021-11-03 15:51:22 [NOTICE] [configure] 300 out of 2460 done.
2021-11-03 15:52:12 [NOTICE] [configure] 400 out of 2460 done.
...
2021-11-03 16:06:20 [NOTICE] [configure] 3000 out of 3300 done.
2021-11-03 16:06:31 [NOTICE] [configure] 3100 out of 3300 done.
2021-11-03 16:08:06 [NOTICE] [configure] 3200 out of 3300 done.
2021-11-03 16:08:31 [NOTICE] [configure] System configuration has finished.
  Success!
  * Satellite is running at https://sat01.example.com
      Initial credentials are admin / Passw0rd!

  * To install an additional Capsule on separate machine continue by running:

      capsule-certs-generate --foreman-proxy-fqdn "$CAPSULE" --certs-tar "/root/$CAPSULE-certs.tar"
  * Capsule is running at https://sat01.example.com:9090

  The full log is at /var/log/foreman-installer/satellite.log
Package versions are being locked.
```
Remember that earlier I said that we will use Satellite for DNS services.  After completing the install above, I change the IP address of my server hosting Satellite and rerun the satellite-installer to update the ip address for the --foreman-proxy-dns-server option.
```
satellite-installer --scenario satellite \
--foreman-proxy-dns-server "10.1.10.254"
```

Use the following command to find the name of the Satellite server you just updated.
```
# hammer proxy list
```

See which services are configured on your Satellite server.  We want to verify that the DNS and DHCP services are enabled.
```
# hammer proxy info --name sat01.example.com
```

If services such as DNS or DHCP are not part of the output from the previous command, try refreshing the Satellite features.
```
# hammer proxy refresh-features --name sat01.example.com
```
 

### Login into the Satellite console  

We can now launch and login to the Satellite console by entering [http://sat01.example.com](http://sat01.example.com) for the Satellite url.  Satellite will redirect the browser to Satellite's secure login page.  You will need to accept Satellite's certificate for your browser.  

For this example we are using a local login.  For production work you will want to integrate your directory service with Satelllite. Enter the user id and password and click Login button.  

![Click Login button](/images/sat01.png)  

You are now at the Satellite home screen.  

![Satellite Home Srceen](/images/sat02.png)  



## References  
[Installing Satellite Server from a Connected Network](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.9/html/installing_satellite_server_from_a_connected_network/index)   
[Simple Content Access](https://access.redhat.com/articles/simple-content-access)  
[Provisioning VMWare using userdata via Satellite 6.3-6.6](https://access.redhat.com/blogs/1169563/posts/3640721)  
[Understanding Red Hat Content Delivery Network Repositories and their usage with Satellite 6](https://access.redhat.com/articles/1586183)

