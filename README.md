# usbip 

This tutorial explains how to set up a usbip server and client. In my case the server is a raspberry pi and the client a ubuntu vm running in a proxmox environment. By running this setup, the vm can circle through different hosts but keep talking to a usb device (for instance a deconz conbee 2 zigbee stick). 

## server (raspbian)

The following two services will run the usbip daemon and for each bound device one other service. If a device gets attached and detached at a client, the server will re-bind and make the devices available again.

First, we install usbip.

```
sudo -s
apt-get install usbip
modprobe usbip_host
echo 'usbip_host' >> /etc/modules
```

### run daemon

Create the following service to run the usbip daemon:
```
nano /lib/systemd/system/usbipd.service
```
with contents:
```
[Unit]
Description=usbip host daemon
After=network.target

[Service]
Type=forking
ExecStart=/usr/sbin/usbipd -D

[Install]
WantedBy=multi-user.target
```

Now we start the daemon service by:
```
systemctl --system daemon-reload
systemctl enable usbipd.service
systemctl start usbipd.service
```

### run a bind

Create the following service to bind any device:
```
nano /lib/systemd/system/usbip-bind@.service
```
with contents:
```
[Unit]
Description=USB-IP Binding on bus id %I
After=network-online.target usbipd.service
Wants=network-online.target
Requires=usbipd.service

[Service]
Type=simple
ExecStart=/usr/sbin/usbip bind -b %i
RemainAfterExit=yes
ExecStop=/usr/sbin/usbip unbind -b %i
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

To now bind a device, enable and start a service like, where `1-1.3` is one device ID from `usbip list -l`:
```
systemctl enable usbip-bind@1-1.3.service
systemctl start usbip-bind@1-1.3.service
```


## client (ubuntu 20)

Also here, first we install usbip:
```
sudo -s
apt-get install linux-tools-generic -y
modprobe vhci-hcd
echo 'vhci-hcd' >> /etc/modules
```

Then we create a generic client service to bind multiple devices, each with one service.

Create a new file `nano /lib/systemd/system/usbip@.service`
with the following content (and replace `192.168.111.250` by our raspberry pi IP address): 
```
[Unit]
Description=usbip client
After=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/sh -c "/usr/lib/linux-tools/$(uname -r)/usbip attach -r 192.168.111.250 -b %i"
ExecStop=/bin/sh -c "/usr/lib/linux-tools/$(uname -r)/usbip detach --port=$(/usr/lib/linux-tools/$(uname -r)/usbip port | grep '<Port in Use>' | sed -E 's/^Port ([0-9][0-9]).*/\1/')"

[Install]
WantedBy=multi-user.target
```

and enable and start the service: 
```
systemctl enable usbip@1-1.3
systemctl start usbip@1-1.3
```

### run jupyter lab

```
import paramiko
import re
# Tạo một SSH client
ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
```
```
# Kết nối đến server
raspi_ip = 'usbip05.local'
ssh.connect(raspi_ip, port=22, username='pi', password='0xxxxxx70')

# Thực hiện một lệnh trên server
stdin, stdout, stderr = ssh.exec_command('sudo usbip list -l|grep 1414')
result = stdout.read().decode()
busids = re.findall(r'busid\s([0-9\-\.]+)', result)
print("Total busids found:", len(busids))

# Join the found busids into a single string separated by commas
result = ','.join(busids)
for busid in busids:
    print('systemctl enable usbip-bind@'+busid+'.service')
    print('systemctl start usbip-bind@'+busid+'.service')
```

