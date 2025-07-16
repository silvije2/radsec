# Radsec on Mikrotik with FreeRadius howto

## Certificates

We'll need some certificates. First let's create our own CA:
```
openssl req -x509 -sha256 -days 3650 -nodes -newkey rsa:4096 -keyout my_root_CA.key -out my_root_CA.pem -subj "/C=HR/ST=Hrvatska/L=Zagreb/O=MyOrg/OU=MyNet/CN=*.myorg.hr"
```

Then we'll need certificates for both our server running freeradius and our Mikrotik device.
Let's create server certificate. First create server_ext.cnf file with following entries:
```
[ server_cert_ext ]
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = IP:192.168.1.1
```
Use actual IP address of your server in file above and also in commands bellow, instead of 192.168.1.1.
```
openssl genrsa -out server.key 4096

openssl req -new -key server.key \
    -out server.csr \
    -subj "/C=HR/ST=Hrvatska/L=Zagreb/O=MyOrg/CN=192.168.1.1/emailAddress=admin@myorg.hr"

openssl x509 -req -in server.csr \
    -CA my_root_CA.pem \
    -CAkey my_root_CA.key \
    -CAcreateserial \
    -extfile server_ext.cnf \
    -extensions server_cert_ext \
    -out server.crt -days 999
```

Now let's create certificate for our Mikrotik device. Similarly we need client_ext.cnf file with following entries:
```
[ client_cert_ext ]
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
subjectAltName = IP:172.16.1.1
```
Use actual IP address of your Mikrotik device in file above and also in commands bellow, instead of 172.16.1.1.
```
openssl genrsa -out mikrotik.key 4096

openssl req -new -key mikrotik.key \
    -out mikrotik.csr \
    -subj "/C=HR/ST=Hrvatska/L=Zagreb/O=MyOrg/CN=172.16.1.1/emailAddress=admin@myorg.hr"

openssl x509 -req -in mikrotik.csr \
    -CA my_root_CA.pem \
    -CAkey my_root_CA.key \
    -CAcreateserial \
    -extfile client_ext.cnf \
    -extensions client_cert_ext \
    -out mikrotik.crt -days 999
```

## Mikrotik RouterOS

Let's update RouterOS to version 7.19.3 at least, older versions will not work.
Now let's copy mikrotik.crt, mikrotik.key and my_root_CA.pem to Mikrotik device and then execute:
```
/certificate import file-name=my_root_CA.pem passphrase=""
/certificate import file-name=mikrotik.crt passphrase=""
/certificate import file-name=mikrotik.key passphrase=""
```

Now we have our certificates ready. Let's configure RADSEC:
Use actual IP addresses of your Mikrotik device and freeradius server in commands bellow.
```
/radius
add address=192.168.1.1 certificate=mikrotik.crt_0 protocol=radsec require-message-auth=no service=login src-address=172.16.1.1 timeout=300ms
/user aaa set default-group=full exclude-groups=full use-radius=yes
```

also set NTP servers:
```
/system ntp client
set enabled=yes
/system ntp client servers
add address=<<address of your NTP server>>
```

## FreeRadius

If we have working freeradius configuration we only need to adjust few things.

Edit configuration file freeradius/sites-available/tls
In tls section edit paths to private_key_file, certificate_file and ca_file
by entering server.key, server.crt and my_root_CA.pem filenames with full path to them.

Next let's add our Mikrotik client to 'clients radsec' section
```
client  172.16.1.1 {
        ipaddr = 172.16.1.1
        secret = radsec
        nastype = other
        proto = tls
        virtual_server = default
}
```
You probably already have virtual server for your mikrotik so instead of default use that.
If not, google it, I'm too lazy to go to every detail.

Then let's symlink this tls file to sites-enabled and restart freeradius service.

That should be it.
