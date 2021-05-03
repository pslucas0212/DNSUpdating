# DNS Dynaamic Updating
Notes on setting up remote DNS updating with nsupdate

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



- Update bind bind-utils on the Satellite Server

      # yum install bind bind-utils
    
- Copy and prepare the rndc.key from the server running named

      # scp root@ns02.example.com:/etc/rndc.key /etc/rndc.key
      # restorecon -v /etc/rndc.key
      # chown -v root:named /etc/rndc.key
      # chmod -v 640 /etc/rndc.key
      
- Command to update forward zone

      # echo -e "server 10.1.10.254\n update add 10.1.10.10.in-addr.arpa. 300 PTR atest.example.com\n send\n" | nsupdate -k /etc/rndc.key
    
