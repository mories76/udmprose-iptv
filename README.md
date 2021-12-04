# IPTV on the UniFi Dream Machine PRO SE

This document, altought highly experimental, will describe how to get podman running on the UDM PRO SE.  
Please be warned, I am no expert at this field.
Please let me know if you have any advice, a comment, a 👍 or anything else.

There are two project which this procedure heavilly relies on.  
- https://github.com/boostchicken/udm-utilities
- https://github.com/fabianishere/udm-iptv

I assume you have a fair bit of understanding on the concept of routed IPTV.

# Requirements

Your UDM PRO SE needs run a kernel which supports igmpproxy / multicastrouting.  
At the time of writing (early december 2021) you need early access firmware 2.3.7  
https://community.ui.com/releases/UniFi-OS-Dream-Machine-SE-2-3-7/2cf1632b-bcf6-4b13-a61d-f74f1e51242c?page=1

Make sure you read and understand the readme at https://github.com/fabianishere/udm-iptv.  
Especially the sections, "Setting up Internet Connection" and "Configuring Internal LAN".

Prepare the variables you need
| Environmental Variable | Description | Default |
| ------------------------|----------- |---------|
| IPTV_WAN_INTERFACE      | Interface on which IPTV traffic enters the router | eth8 (on UDM Pro) or eth4 (on UDM) |
| IPTV_WAN_RANGES         | IP ranges from which the IPTV traffic originates (separated by spaces) | 213.75.0.0/16 217.166.0.0/16 |
| IPTV_WAN_VLAN           | ID of VLAN which carries IPTV traffic (use 0 if no VLAN is used) | 4 |
| IPTV_WAN_VLAN_INTERFACE | Name of the VLAN interface to be created | iptv |
| IPTV_WAN_DHCP_OPTIONS   | [DHCP options](https://busybox.net/downloads/BusyBox.html#udhcpc) to send when requesting an IP address | -O staticroutes -V IPTV_RG |
| IPTV_LAN_INTERFACES     | Interfaces on which IPTV should be made available | br0 |

# Download podman

In the udm-utilities repo there is a section with UDMSE Podman workflow under "Actions" -> "Workflow" -> "UDMSE Podman".  
You have to be logged in to github to see the artifact.  
https://github.com/boostchicken/udm-utilities/actions/workflows/podman-udmse.yml  
Download udmse-podman-install.zip from the artifacts of the latest workflow.  
I haven't found a way the copy a download link to the latest artifact, so for now download it on your local machine.

# Copy udmse-podman-install.zip with scp to UDM PRO SE

- Use scp cli, or a gui to copy the file to /tmp  
  for example on macos: 
  ```
  scp ~/Downloads/udmse-podman-install.zip root@192.168.1.1:/tmp/
  ```
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

download the following files with the contents from this repo
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

# Set environment variables and start container
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
