# DNS Dynaamic Updating
Notes on setting up remote DNS updating with nsupdate

- Update bind bind-utils on the Satellite Server
      # yum install bind bind-utils
    
- Copy and prepare the rndc.key from the server running named
      # scp root@ns02.example.com:/etc/rndc.key /etc/rndc.key
      # restorecon -v /etc/rndc.key
      # chown -v root:named /etc/rndc.key
      # chmod -v 640 /etc/rndc.key
      
- Update DNS record
      # echo -e "zone example.com.\n server 10.1.10.254\n update add aaa.virtual.lan 3600 IN A 10.1.10.245\n send\n" | nsupdate -k /etc/rndc.key
    
