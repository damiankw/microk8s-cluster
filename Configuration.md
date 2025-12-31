Configure node

# Install microk8s and add to cluster
From Host ``

# Configure netdata
- Install: ```wget -O /tmp/netdata-kickstart.sh https://get.netdata.cloud/kickstart.sh && sh /tmp/netdata-kickstart.sh```
- Configure: ```sudo /etc/netdata/edit-config stream.conf```
  - Update host, api, etc.

# Set up nano/pico
- Update editor in bash: ```echo 'export EDITOR=pico' >> ~/.bashrc```
- Import source: ```source ~/.bashrc```

# Set up password requirement
- Open visudo: ```sudo visudo```
- Updated the %sudo line to become: ```%sudo   ALL=(ALL:ALL) NOPASSWD:ALL```
