# DNS Dynamic Updating

Update in progress - 2022-03-17

**Note:** named is running on a RHEL 8.5 server updated in March 2022. For this example the subnet is 10.1.10.0/24 and domain is example.com

### Pre-Reqs
Create a RHEL 8.5 VM to provide DDNS and DHCP services.  The VM was sized with 2 vCPUS, 4GB RAM and 100GB "local" drive.  Note: For this example I have enabled Simple Content Access (SCA) on the Red Hat Customer portal and do not need to attach a subscription to the RHEL repositories.  After you have created and started the RHEL 8.5 VM, we will ssh to the RHEL VM and work from the command line.

For this lab environment I chose ns02.example.com for the hostname of the server. 

Register the Server to Red Hat Subscription Management service.
```
# sudo subscription-manager register --org=<org id> --activationkey=<activation key>
```
You can verify the registration with the following command.
```
# sudo subscription-manager status
```    
#### Enabled repositories  

We will want the following two RHEL 8 repositoires enabled on this system:
- rhel-8-for-x86_64-baseos-rpms
- rhel-8-for-x86_64-appstream-rpms

Verify that repositories are enabled.  
```    
# sudo subscription-manager repos --list-enabled
+----------------------------------------------------------+
    Available Repositories in /etc/yum.repos.d/redhat.repo
+----------------------------------------------------------+
Repo ID:   rhel-8-for-x86_64-baseos-rpms
Repo Name: Red Hat Enterprise Linux 8 for x86_64 - BaseOS (RPMs)
Repo URL:  https://cdn.redhat.com/content/dist/rhel8/$releasever/x86_64/baseos/o
           s
Enabled:   1

Repo ID:   rhel-8-for-x86_64-appstream-rpms
Repo Name: Red Hat Enterprise Linux 8 for x86_64 - AppStream (RPMs)
Repo URL:  https://cdn.redhat.com/content/dist/rhel8/$releasever/x86_64/appstrea
           m/os
Enabled:   1
```          

#### Update the RHEL 8.5 VM and finish server setup
Install all the latest patches on your RHEL 8.5 Server VM
```
# sudo yum -y update
Updating Subscription Management repositories.
Red Hat Enterprise Linux 8 for x86_64 - BaseOS   27 MB/s |  45 MB     00:01    
Red Hat Enterprise Linux 8 for x86_64 - AppStre  26 MB/s |  39 MB     00:01    
Dependencies resolved.
...
Complete!
```
 
I would also recommend registering this server to Insights.  
```
# sudo insights-client --enable
```


## Install named and dhcpd

We will install named, the bind utilities, the dns caching sever and dhcpd.
```
# sudo yum -y install bind* caching* dhcp*
...
Complete!
```


### Notes on setting up dynamic DNS with nsupdate

- Update named.conf.   Add the following lines:
```
include "/etc/rndc.key";

acl "rndc-users" {
     10.1.10.0/24;
};

controls {
      // localhost - default key
      inet 127.0.0.1 allow {localhost;};
      inet * port 7766 allow {"rndc-users";} keys { "rndc-key"; };
};
```

- Add the following line to both forward and reverse zone info in named.conf
```
allow-update {key rndc-key;};
```

- You will need bind installed to use nsupdate.  Instaoo or update bind-utils on the client Server as needed

      # yum list installed | grep bind-utils
      # yum install bind-utils
    
- Copy and prepare the rndc.key from the server running named

      # scp root@ns02.example.com:/etc/rndc.key /etc/rndc.key
      # restorecon -v /etc/rndc.key
      # chown -v root:named /etc/rndc.key
      # chmod -v 640 /etc/rndc.key
      
- Test updates to the forward zone (add -d to nsupdate comand for debug: nsupdate -d -k ...)

      # echo -e "zone example.com.\n server 10.1.10.254\n update add atest.example.com 3600 IN A 10.1.10.10\n send\n" | nsupdate -k /etc/rndc.key
      # nslookup atest.example.com
      # echo -e "zone example.com.\n server 10.1.10.254\n update delete atest.example.com 3600 IN A 10.1.10.10\n send\n" | nsupdate -k /etc/rndc.key
      
- Test updates to reverse zone (add -d to nsupdate comand for debug: nsupdate -d -k ...)
     
      # echo -e "zone 10.1.10.in-addr.arpa.\n server 10.1.10.254\n update add 10.10.1.10.in-addr.arpa. 300 PTR atest.example.com\n send\n" | nsupdate -k /etc/rndc.key
      # nslookup 10.1.10.10
      # dig +short -x 10.1.10.10
      # echo -e "zone 10.1.10.in-addr.arpa.\n server 10.1.10.254\n update delete 10.10.1.10.in-addr.arpa. 300 PTR atest.example.com\n send\n" | nsupdate -k /etc/rndc.key
     
- For convience - add new dns entry forward and reverse 
  
      # echo -e "zone example.com.\n server 10.1.10.254\n update add atest.example.com 3600 IN A 10.1.10.10\n send\n" | nsupdate -k /etc/rndc.key
      # echo -e "zone 10.1.10.in-addr.arpa.\n server 10.1.10.254\n update add 10.10.1.10.in-addr.arpa. 300 PTR atest.example.com\n send\n" | nsupdate -k /etc/rndc.key
      
- For convience - delete dns entry forward and reverse 

      # echo -e "zone example.com.\n server 10.1.10.254\n update delete atest.example.com 3600 IN A 10.1.10.10\n send\n" | nsupdate -k /etc/rndc.key
      # echo -e "zone 10.1.10.in-addr.arpa.\n server 10.1.10.254\n update delete 10.10.1.10.in-addr.arpa. 300 PTR atest.example.com\n send\n" | nsupdate -k /etc/rndc.key
      
      
### References
- [How to configure the BIND DNS service](https://access.redhat.com/solutions/40683)
- [How to configure Dynamic DNS Server on AlmaLinux / Rocky Linux](https://www.techbrown.com/configure-dynamic-dns-server-almalinux-rocky-linux/)
