# DNS Dynamic Updating

Updated 2021-05-03  
**Note:** named is running on a RHEL 8.3 server updated in April 2021 and the test client servers is also running RHEL 8.3  
For this example the subnet is 10.1.10.0/24 and domain is example.com


## Notes on setting up dynamic DNS with nsupdate

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
      # echo -e "zone 10.1.10.in-addr.arpa.\n server 10.1.10.254\n update add 10.10.1.10.in-addr.arpa. 300 PTR atest.example.com\n send\n" | nsupdate -k /etc/rndc.key
     
- For convience - add new dns entry forward and reverse   
      # echo -e "zone example.com.\n server 10.1.10.254\n update add atest.example.com 3600 IN A 10.1.10.10\n send\n" | nsupdate -k /etc/rndc.key
      # echo -e "zone 10.1.10.in-addr.arpa.\n server 10.1.10.254\n update add 10.10.1.10.in-addr.arpa. 300 PTR atest.example.com\n send\n" | nsupdate -k /etc/rndc.key
      
      
      
     
