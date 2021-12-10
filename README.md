# IPTV on the UniFi Dream Machine PRO SE & UniFi Dream Router

This document, altought highly experimental, will describe how to get podman running on the UDM PRO SE & UDR.  
Please be warned, I am no expert at this field.
Please let me know if you have any advice, a comment, a 👍 or anything else.

There are two project which this procedure heavilly relies on.  
- https://github.com/boostchicken/udm-utilities
- https://github.com/fabianishere/udm-iptv

I assume you have a fair bit of understanding on the concept of routed IPTV.

# Requirements

Your UDM PRO SE or UDR needs run a kernel which supports igmpproxy / multicastrouting.  
At the time of writing (early december 2021) you need early access firmware 2.3.7  
https://community.ui.com/releases/UniFi-OS-Dream-Machine-SE-2-3-7/2cf1632b-bcf6-4b13-a61d-f74f1e51242c?page=1

Make sure you read and understand the readme at https://github.com/fabianishere/udm-iptv.  
Especially the sections, "Setting up Internet Connection" and "Configuring Internal LAN".

Assign the separately created network to the intended switchport via the GUI. Example: You created under networks a new network with the name "LAN-IPTV" and its subnet is 192.168.2.0/24. You will use physical port 4 on your device, open your UDMSE/UDR under Unifi devices in the portal. Open the settings of port 4, then assign port profile "LAN-IPTV" as port profile and save.

Additional information: Retrieving the "IPTV_LAN_INTERFACES" value (Can be br# which # is corresponding number) specifically for your set-up can be achieved by configuring your "LAN" in the web-console as described in the article mentioned above and note the intended "subnet" down. Start a SCP CLI session (Eg PuTTy) to your device and perform command
```
ifconfig
```
Scroll through the output list and find your subnet under the "inet" value. Note this down for further steps (You need this in "environment variables" section).

Prepare the variables you need
| Environmental Variable | Description | Default |
| ------------------------|----------- |---------|
| IPTV_WAN_INTERFACE      | Interface on which IPTV traffic enters the router | eth8 (on UDM Pro) or eth4 (on UDM) |
| IPTV_WAN_RANGES         | IP ranges from which the IPTV traffic originates (separated by spaces) | 213.75.0.0/16 217.166.0.0/16 |
| IPTV_WAN_VLAN           | ID of VLAN which carries IPTV traffic (use 0 if no VLAN is used) | 4 |
| IPTV_WAN_VLAN_INTERFACE | Name of the VLAN interface to be created | iptv |
| IPTV_WAN_DHCP_OPTIONS   | [DHCP options](https://busybox.net/downloads/BusyBox.html#udhcpc) to send when requesting an IP address | -O staticroutes -V IPTV_RG |
| IPTV_LAN_INTERFACES     | Interfaces on which IPTV should be made available | br0 |

# Download and copy podman installation archive to UDM PRO SE

These steps needs to be performed on your computer.

- In the udm-utilities repo there is a section with UDMSE Podman workflow under "Actions" -> "Workflow" -> "UDMSE Podman".  
  You have to be logged in to github to see the artifact.  
  https://github.com/boostchicken/udm-utilities/actions/workflows/podman-udmse.yml  
  On your computer download udmse-podman-install.zip from the artifacts of the latest workflow.  
  I haven't found a way the copy a download link to the latest artifact, so for now download it on your local machine.

- Use scp cli (Eg PuTTy), or a GUI to copy the file udmse-podman-install.zip the UDM PRO SE in the folder /tmp  

  for example on macos: 
```
scp ~/Downloads/udmse-podman-install.zip root@192.168.1.1:/tmp/
```
  
# Extract zip file

Alle steps from here one are done through ssh on your UDM PRO SE

- Extract the zip file to /tmp/podman
```
unzip /tmp/udmse-podman-install.zip -d /tmp/podman
```
- This is a nested zip file, so extract the last one to root
```
unzip /tmp/podman/podman-install.zip -d /
```

# Download cni drivers

- Use the following script from boostchickens udm-utilities repo.  
  The cni drivers are insalled in /opt/cni/bin
```
sh -c "$(curl -s https://raw.githubusercontent.com/boostchicken/udm-utilities/master/cni-plugins/05-install-cni-plugins.sh)"
```

# Create additional config files in /etc/containers

Download the following files with the contents from this repo
- /etc/containers/policy.json
- /etc/containers/registries.conf
- /etc/containers/storage.conf
```
wget https://raw.githubusercontent.com/mories76/udmprose-iptv/main/policy.json -P /etc/containers
wget https://raw.githubusercontent.com/mories76/udmprose-iptv/main/registries.conf -P /etc/containers
wget https://raw.githubusercontent.com/mories76/udmprose-iptv/main/storage.conf -P /etc/containers
```

# Change /etc/containers.conf
Add log_diver value to [containers] section by using this command
```
sed -i '/^log_size_max=.*/i log_driver="journald"' /etc/containers/containers.conf
```

The first section of /etc/containers/containers.conf should look something like this

```
[containers]
cgroups="no-conmon"
pidns="private"
pids_limit=0
log_driver="journald"
log_size_max=104857600
```

# Set environment variables, create and start container
Values for UDMPSE usually are:
```
export IPTV_WAN_INTERFACE="eth8"
export IPTV_WAN_RANGES="213.75.0.0/16 217.166.0.0/16"
export IPTV_WAN_VLAN="4"
export IPTV_WAN_VLAN_INTERFACE="iptv"
export IPTV_WAN_DHCP_OPTIONS="-O staticroutes -V IPTV_RG"
export IPTV_LAN_INTERFACES="br2"
export IPTV_LAN_RANGES=""
export IPTV_IGMPPROXY_ARGS=""

podman run --network=host --privileged \
    --name iptv -i -d --restart on-failure:5 \
    -e IPTV_WAN_INTERFACE="$IPTV_WAN_INTERFACE" \
    -e IPTV_WAN_RANGES="$IPTV_WAN_RANGES" \
    -e IPTV_WAN_VLAN="$IPTV_WAN_VLAN" \
    -e IPTV_WAN_DHCP_OPTIONS="$IPTV_WAN_DHCP_OPTIONS" \
    -e IPTV_LAN_INTERFACES="$IPTV_LAN_INTERFACES" \
    -e IPTV_LAN_RANGES="" \
    fabianishere/udm-iptv $IPTV_IGMPPROXY_ARGS
```

Values for the UDR usually are:
```
export IPTV_WAN_INTERFACE="eth4"
export IPTV_WAN_RANGES="213.75.0.0/16 217.166.0.0/16"
export IPTV_WAN_VLAN="4"
export IPTV_WAN_VLAN_INTERFACE="iptv"
export IPTV_WAN_DHCP_OPTIONS="-O staticroutes -V IPTV_RG"
export IPTV_LAN_INTERFACES="br2"
export IPTV_LAN_RANGES=""
export IPTV_IGMPPROXY_ARGS=""

podman run --network=host --privileged \
    --name iptv -i -d --restart on-failure:5 \
    -e IPTV_WAN_INTERFACE="$IPTV_WAN_INTERFACE" \
    -e IPTV_WAN_RANGES="$IPTV_WAN_RANGES" \
    -e IPTV_WAN_VLAN="$IPTV_WAN_VLAN" \
    -e IPTV_WAN_DHCP_OPTIONS="$IPTV_WAN_DHCP_OPTIONS" \
    -e IPTV_LAN_INTERFACES="$IPTV_LAN_INTERFACES" \
    -e IPTV_LAN_RANGES="" \
    fabianishere/udm-iptv $IPTV_IGMPPROXY_ARGS
```

# Take a breath

There might be a chance that everything is up and running.  
Is everything working ?  
Then stop the container and proceed with the last steps.  
```
podman stop iptv
```

# Run podman as a service

Podman can create a systemd unit file.
By creating a service for podman and the iptv container, the service can be started at boot time.
Perhaps create the service file in /etc/systemd/system so it can survive firmware upgrades.
IDEA: Maybe have the service check for existence of the podman binaries and other prerequisites ?

https://docs.podman.io/en/latest/markdown/podman-generate-systemd.1.html
https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files

Create a systemd service unit file so that the iptv container can start at boot time 
```
podman generate systemd --restart-policy=always iptv > /etc/systemd/system/iptv.service
```

Add configuration parameters to the unit file (UDMPSE, when using physical port 8 on your device)
```
sed -i '/^Restart=always/i Environment=IPTV_WAN_INTERFACE="eth8"' /etc/systemd/system/iptv.service
sed -i '/^Restart=always/i Environment=IPTV_WAN_RANGES="213.75.0.0/16 217.166.0.0/16"' /etc/systemd/system/iptv.service
sed -i '/^Restart=always/i Environment=IPTV_WAN_VLAN="4"' /etc/systemd/system/iptv.service
sed -i '/^Restart=always/i Environment=IPTV_WAN_VLAN_INTERFACE="iptv"' /etc/systemd/system/iptv.service
sed -i '/^Restart=always/i Environment=IPTV_WAN_DHCP_OPTIONS="-O staticroutes -V IPTV_RG"' /etc/systemd/system/iptv.service
sed -i '/^Restart=always/i Environment=IPTV_LAN_INTERFACES="br2"' /etc/systemd/system/iptv.service
sed -i '/^Restart=always/i Environment=IPTV_LAN_RANGES=""' /etc/systemd/system/iptv.service
sed -i '/^Restart=always/i Environment=IPTV_IGMPPROXY_ARGS=""' /etc/systemd/system/iptv.service
```

Add configuration parameters to the unit file (UDR, when using physical port 4 on your device)
```
sed -i '/^Restart=always/i Environment=IPTV_WAN_INTERFACE="eth4"' /etc/systemd/system/iptv.service
sed -i '/^Restart=always/i Environment=IPTV_WAN_RANGES="213.75.0.0/16 217.166.0.0/16"' /etc/systemd/system/iptv.service
sed -i '/^Restart=always/i Environment=IPTV_WAN_VLAN="4"' /etc/systemd/system/iptv.service
sed -i '/^Restart=always/i Environment=IPTV_WAN_VLAN_INTERFACE="iptv"' /etc/systemd/system/iptv.service
sed -i '/^Restart=always/i Environment=IPTV_WAN_DHCP_OPTIONS="-O staticroutes -V IPTV_RG"' /etc/systemd/system/iptv.service
sed -i '/^Restart=always/i Environment=IPTV_LAN_INTERFACES="br2"' /etc/systemd/system/iptv.service
sed -i '/^Restart=always/i Environment=IPTV_LAN_RANGES=""' /etc/systemd/system/iptv.service
sed -i '/^Restart=always/i Environment=IPTV_IGMPPROXY_ARGS=""' /etc/systemd/system/iptv.service
```

Enable service to start at boot
```
systemctl enable iptv
```

Check if previous command succeeded, look for iptv.service
```
systemctl list-unit-files --type service
```

Start service

```
systemctl start iptv
```

Check status of service
```
systemctl status iptv
```

# Firmware updates

*To be refined*  

After a firmware update, certain parts of the filesystem are being reset due to overlayfs.  
The /etc folder remains but the /opt foler is being reset. This means that the podman binaries in /usr/bin and /usr/libexec are missing. The configuration files in /etc/containers are still there.
I am not sure yet about the /var/lib/containers folder. This is the folder where the containers images are kept.  

- Repeat the step from [Download and copy podman installation archive to UDM PRO SE](#download-and-copy-podman-installation-archive-to-udm-pro-se)
- Repeat the step from [Extract zip file](#extract-zip-file)
  But when extracing the second file do not replace the file /etc/containers/containers.conf when asked for
- Clear the storage/cache older
```
rm -rf /var/lib/containers/*
```
- Repeat the steps from [Set environment variables and start container](#set-environment-variables-and-start-container)
  

