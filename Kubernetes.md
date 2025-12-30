The IDEA!

The idea here is to install and run a Kubernetes cluster on HP EliteDesk Mini's. These mini's pack a punch when it comes to the hardware inside of them. I'm using a mixture of the following hardware for my setup:

- HP EliteDesk 800 G5 Mini 7YY06PA [i7-9700T, 2 x 256GB NVMe, 2 x 8GB RAM]
- HP EliteDesk 800 G5 Mini 7YX38PA [i5-9500T, 2 x 256GB NVMe, 2 x 8GB RAM]

I want to configure these with local disk redundancy in case of disk failure, so instead of losing a node in the cluster I can gracefully stop it and replace the disk before there's a real issue.

# Get hardware ready
Before going onto the operating system I first started with the system itself. The basics of this is installing the disks in the order I wanted them, I always install disks using serial number, lowest to highest, that way if I need to do diagnosis later it's easy.

I also run a firmware update on BIOS, with HP machines this is too simple, just go into the BIOS and update firmware, voila.

Check that the system recognises the disks and the memory and you're typically golden. For extra measure I could have run some checks on these using the HP utilities, but I didn't feel the need.

# Install Ubuntu
## Partitioning
When installing Ubuntu I needed to go through a bit of a process to get the disks the way I wanted, I had to create the following partitions on the disks:

| Device  | Partition | Size    | Type   | Mount Point |
|---------|-----------|---------|--------|-------------|
| NVMe01  | 01        | 1GB     | FAT32  | /boot/efi   |
| NVMe01  | 02        | 2GB     | RAID1  | /boot       |
| NVMe01  | 03        | ~230GB  | RAID1  | /           |
|         |           |         |        |             |
| NVMe02  | 01        | 1GB     | FAT32  | /boot/efi   |
| NVMe02  | 02        | 2GB     | RAID1  | /boot       |
| NVMe02  | 03        | ~230GB  | RAID1  | /           |

How I achieved this was to do a bit of manual configuration of the disks. At the 'Guided storage configuration' step of installation select 'Custom storage layout' and then go through the following steps:

Create the partitions:
- Select disk 1, whack enter and select Reformat > Reformat
- Select disk 1 free space, whack enter and select Add GPT Partition > Size (1G); Format (Leave unformatted) > Create
- Select disk 1 free space, whack enter and select Add GPT Partition > Size (2G); Format (Leave unformatted) > Create
- Select disk 1 free space, whack enter and select Add GPT Partition > Size (leave empty); Format (Leave unformatted) > Create
- Select disk 2, whack enter and select Reformat > Reformat
- Select disk 2 free space, whack enter and select Add GPT Partition > Size (1G); Format (Leave unformatted) > Create
- Select disk 2 free space, whack enter and select Add GPT Partition > Size (2G); Format (Leave unformatted) > Create
- Select disk 2 free space, whack enter and select Add GPT Partition > Size (leave empty); Format (Leave unformatted) > Create

Create the Array (RAID1):
- Select Create software RAID (md), whack enter and select the two 2G partitions from the two disks > Create
- Select Create software RAID (md), whack enter and select the two ~230G partitions from the two disks > Create
You might have noticed we didn't create an array out of the 1GB partition, this is going to be our boot partition the system picks up on, so it needs to be separated so the system can read it.

Create the file system mounts:
- Select md0 free space, whack enter and select Add GPT Partition > Mount (/boot) > Create
- Select md0 free space, whack enter and select Add GPT Partition > Mount (/) > Create

This will create the /boot and / file systems. If you want to separate anything else, you can do it here.

Once you're done, you should have a file system table that looks something like what's below, you can go ahead and whack Done!

| MOUNT POINT | SIZE  | TYPE      | DEVICE TYPE                      |
|-------------|-------|-----------|----------------------------------|
| /           | ~230G | new ext4  | new partition of software RAID 1 |
| /boot       | 2G    | new ext4  | new partition of software RAID 1 |
| /boot/efi   | 1G    | new fat32 | new partition of local disk      |

The rest of the installation do as normal, however during the 'Featured server snaps' I selected to install microk8s, as that's what I want to build my Kubernetes cluster with.

I also recommend that if you haven't already, that you use Github to manage your SSH keys. When you enable OpenSSH Server on Ubuntu installation you can then select to download them from Github and voila, instant access.

## Configure second /boot/efi
After installing Ubuntu I needed to go through and create the second boot partition on the second disk, this will allow me to boot from the second disk if the first disk buggered up.

Check to make sure all of your partitions are in order first, by doing: lsblk

You should see something like this, where nvme0n1p1 has /boot/efi, and nvme0n1p1 has nothing.

```
NAME        MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
nvme0n1     259:0    0 238.5G  0 disk
├─nvme0n1p1 259:2    0     1G  0 part  /boot/efi       << the /boot/efi, this is what the system boots from
├─nvme0n1p2 259:3    0     1G  0 part
├─nvme0n1p3 259:4    0     3G  0 part
│ └─md0       9:0    0     3G  0 raid1
│   └─md0p1 259:9    0     3G  0 part  /boot
└─nvme0n1p4 259:5    0 233.4G  0 part
  └─md1       9:1    0 233.3G  0 raid1
    └─md1p1 259:10   0 233.3G  0 part  /
nvme1n1     259:1    0 238.5G  0 disk
├─nvme1n1p1 259:6    0     1G  0 part                  << the second boot but the system doesn't know how to boot here
├─nvme1n1p2 259:7    0     3G  0 part
│ └─md0       9:0    0     3G  0 raid1
│   └─md0p1 259:9    0     3G  0 part  /boot
└─nvme1n1p3 259:8    0 234.5G  0 part
  └─md1       9:1    0 233.3G  0 raid1
    └─md1p1 259:10   0 233.3G  0 part  /
```

Then we need to create the filesystem and copy the existing boot information to it. These commands will do the following:
- Create filesystem fat32 on /dev/nvme1n1p1
- Create a temporary working folder to mount the new filesystem
- Mount the new filesystem
- Copy everything from the live /boot/efi to the new filesystem
- Unmount the filesystem

```
sudo mkfs.vfat -F 32 /dev/nvme1n1p1
sudo mkdir /mnt/efi
sudo mount /dev/nvme1n1p1 /mnt/efi
sudo rsync -av /boot/efi/ /mnt/efi
sudo umount /mnt/efi
```

And now we update our boot manager to include the new one, while we're at it we're going to update the original one so we have a primary / secondary that we can identify in BIOS.

Run:
```
sudo efibootmgr -c -d /dev/nvme1n1 -p 1 -L "Ubuntu Secondary" -l "\EFI\ubuntu\shimx64.efi"
```

Then run:
```
sudo efibootmgr -v
```

This will give us a list of the boot devices, we're confirming that one of them says 'Ubuntu Secondary', and then we're looking for the one that says 'Ubuntu' and identifying the number after Boot, for example 'Boot0005 Ubuntu HD(....'

Delete the old boot loader for disk one using the number (eg, 0005): ```sudo efibootmgr -b 0005 -B```

Now lets create the boot loader with the new name using the first disk: ```sudo efibootmgr -c -d /dev/nvme0n1 -p 1 -L "Ubuntu Primary" -l "\EFI\ubuntu\shimx64.efi"```

Then reboot, make sure it boots.

Then reboot, select the Ubuntu Secondary, make sure it boots.

Voila! You now have a working system with redundant disks.

# Microk8s
I did about three minutes research on this after seeing microk8s every time I install Ubuntu, so I'm a complete professional here, but I decided to use microk8s instead of standard kubernetes due to simplicity, or so Canonical tell me!

It's assumed the rest of this is done from root (sudo -s) so sudo isn't included in the commands.

During my installation of Ubuntu I set a static IP on all of the nodes, lets assume the following:
- NODE01: 192.168.0.71
- NODE02: 192.168.0.72
- NODE04: 192.168.0.74

Yeah, I have three nodes and they are 1, 2, 4. This is because I'm hoping to get another i7 I can whack in position number 3.

## Create a cluster
From node1 do the following, this will create an invitation to the cluster for the other nodes.

```
microk8s add-node
```

You need to do this for each of the other nodes you have in your cluster, each time you do add-node it generates a new code to use on a single node.

So now, I went ahead and did add-node twice, copy/pasted the command and voila, I have a cluster. In this case I want all of my nodes to be primary nodes, not worker nodes, 'cause it's just small. From what I've read if I have a LOT more nodes I might want only a few to be the primary nodes and others to be workers.

To confirm you now have a working cluster, use the command below. This will list you all of the nodes, their status and version. At this point they should all be the same 'cause I just created them.

```
microk8s kubectl get nodes
NAME          STATUS   ROLES    AGE    VERSION
nichmmini01   Ready    <none>   156m   v1.32.9
nichmmini02   Ready    <none>   97s    v1.32.9
nichmmini04   Ready    <none>   45s    v1.32.9
```

Now to get things working across the cluster! Run these from any of the nodes.

Enable some local storage, microk8s enable hostpath-storage

Enable ingress for websites, microk8s enable ingress

Enable the dashboard for WebUI, microk8s enable dashboard

And lets check what is going on, once you run the following command go to the website it tells you (change the ip) and log in with the token it gives you. You can now see the nodes .. and that's just about it at this point!

microk8s dashboard-proxy

## Configuration and .. stuff
I created a ~/config folder under the root folder so I could put all of my yaml configurations in, so I don't lose them (these should probably go in a backed up folder somewhere that's synced across all hosts, but I'll do that later!)

## Ingress, creating a path into the cluster
In order for Kubernetes to accept traffic regardless of which nodes the services are on, lets create a inbound IP address. This can be done for multiple IP's, just enter them in when asked, for the moment I'm just going to do 192.168.0.70, as I reserved the IP for this specific reason.

```
microk8s enable metallb

.. enter in 192.168.0.70-192.168.0.70
```

.. And then i thought that maybe i'm going to need some more ports on this baby for services that can't push through a reverse proxy, so I attempted to add more and couldn't. What I had to do was edit the existing by finding it and editing it.

find the address pool
```
microk8s kubectl get ipaddresspool -n metallb-system
```

edit the pool, add another line with the second set of addresses, i added 192.168.0.80-192.168.0.89
```
microk8s kubectl describe ipaddresspool default-addresspool -n metallb-system
```

After setting this up, I created a load balancer to nginx, as this is the most used function I would say!

```
apiVersion: v1
kind: Service
metadata:
  name: ingress-lb
  namespace: ingress
spec:
  selector:
    name: nginx-ingress-microk8s
  type: LoadBalancer
  loadBalancerIP: 192.168.0.70
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
    - name: https
      protocol: TCP
      port: 443
      targetPort: 443
```

Tested it, and I get a 404, that's probably good considering I haven't set anything up yet!

## configure ssl certificate
i am going to be using letsencrypt for an ssl for my domain *.internal.nictitate.net, and i host this on cloudflare so easy as pie!

I logged onto cloudflare and got myself an api token to the domain i want to update, and i can store this in a kubernetes secret, which is cool

```
microk8s kubectl create secret generic cloudflare-api-token-secret --from-literal=api-token=[my-api-token] --namespace cert-manager
```

and then configure letsencrypt to talk to cloudflare. create a yaml (util-cloudflare.yaml) and apply it

```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: damian@damian.id.au # Use your real email for expiry alerts
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - dns01:
        cloudflare:
          email: damian@damian.id.au
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
```
..
```
microk8s kubectl apply -f util-cloudflare.yaml
```

```
microk8s enable cert-manager
```


## shared storage across all nodes
now i need to set up shared storage across all of the nodes, this will be where all of the apps core data lives, the databses the configurations, etc. this will replicate across all of the nodes in case a node goes down, another node can start working straight away.

For this I want to use Longhorn, it looks like the easiest to use and doesn't require too much resource overhead.

Add iscsi support to the machine, add the Longhorn repository to microk8s, update the repo directory, then install it using default values.
```
apt install open-iscsi

microk8s helm repo add longhorn https://charts.longhorn.io
microk8s helm repo update
microk8s helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --set csi.kubeletRootDir="/var/snap/microk8s/common/var/lib/kubelet" \
  --set defaultSettings.defaultDataPath="/longhorn"
```

And because I am a visual person I want to see the GUI, so I whacked in some ingress config for nginx to pass to longhorn, here it is..

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ingress
  namespace: longhorn-system
  annotations:
    # 1. Enable regular expressions
    nginx.ingress.kubernetes.io/use-regex: "true"
    # 2. Tell it to rewrite everything after /longhorn to /
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      # 3. Use a capture group (.*) to grab everything after the subpath
      - path: /longhorn(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: longhorn-frontend
            port:
              number: 80
```

and it'll look like this!
<img width="1510" height="509" alt="image" src="https://github.com/user-attachments/assets/c22d55e1-eef2-41f7-bb22-b93e16bf3c39" />



