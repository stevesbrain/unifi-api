unifi-api
=========

This fork of the unifi-api project is dedicated to getting that pesky Framed-IP-Address attribute to a Radius SSO agent (such as a Sonicwall). You can find the main branch at: https://github.com/calmh/unifi-api

History
-------

I needed a solution to send the Framed-IP-Address from a Ubiquiti access point. Unfortunately, Ubiquiti did not support this natively and we already decided on this low cost solution over more expensive devices.

At first I created the unifi-return-ip. This returned a value only after the device was connected. I didn't realize this until the next day I walked into the office and wasn't logged in. I tried using the built-in ippool with FreeRADIUS, but it appeared to be on the reply side of things and I needed it to be in the request portion. 

Being a Windows backend, I ran into multiple problems trying to find commands I could run from Linux. I ultimately found winexe which allowed me to run the netsh command. http://download.opensuse.org/repositories/home:/ahajda:/winexe/ or http://sourceforge.net/projects/winexe/files/

My environment's accounting is as follows: NPS -> FreeRADIUS -> SonicWall

The current limitation is when you first connect a device, it takes 5-10 additional seconds to generate the Framed-IP-Address. You also see this when the DHCP lease expires.

### Mysql setup

```
CREATE DATABASE radius;
GRANT ALL ON radius.* TO radius@localhost IDENTIFIED BY "radpass";
FLUSH PRIVILEGES;
USE radius;
CREATE TABLE dhcp_dump (PID INT NOT NULL AUTO_INCREMENT,PRIMARY KEY(PID),IP TEXT,MAC TEXT);
```

### Cron setup

```
0	1	*	*	*	/usr/local/bin/unifi-winexe
```


### FreeRADIUS setup

sites-enabled/default
```
...
pre-proxy {
#The device has connected before
if (`/bin/bash /usr/local/bin/unifi-sql-bash %{Calling-Station-Id}`) {
        update proxy-request {
		Framed-IP-Address := `/bin/bash /usr/local/bin/unifi-sql-bash %{Calling-Station-Id}`
        }
}
else {
update proxy-request {
        Framed-IP-Address := `/usr/bin/python /usr/local/bin/unifi-return-ip -c localhost -u admin -p p4ssw0rd -v v3 -s default -m %{Calling-Station-Id}` 
    }
}

    
...
}
```

proxy.conf
```
realm DEFAULT {
#    authhost        = 10.10.1.1:1812 # Do not use
    accthost        = 10.10.0.1:1813 # SonicWall
    secret          = secret
}
```

### SonicWall install

Users > Settings > Configure SSO > Radius Accounting > Add
```
Client: FreeRadius Server
Shared Secret: secret
Confirm: secret
User-name
No forwarding
(Don't check the "Log user out if no accounting interim updates are received" option)
```
Apply and enjoy!

License
-------

MIT

