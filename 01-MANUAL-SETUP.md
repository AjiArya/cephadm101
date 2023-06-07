# Part 1: Manual Setup Ceph Cluster
# Pre-requisites
## Execute on all hosts
1. Hosts file
```bash
cat<<EOF | sudo tee -a /etc/hosts
# Demo Ceph Cluster #
192.168.10.11 demo-ceph01
192.168.10.12 demo-ceph02
192.168.10.13 demo-ceph03
EOF
```

2. Update & Upgrade Packages
```bash
sudo apt update -y; sudo apt upgrade -y
```

3. Install ceph package
```bash
sudo apt install -y ceph
```

4. Generate and Distribute SSH Key
```bash
# Generate SSH Key from demo-ceph01
ssh-keygen

# Get SSH Public key from demo-ceph01
cat ~/.ssh/id_rsa.pub

# Authorize demo-ceph01's SSH Key to all hosts
cat<<EOF >> ~/.ssh/authorized_keys
<SSH_PUB_KEY_CONTENT>
EOF
```

# Guide
## Execute on demo-ceph01
1. Create initial config file
```bash
# Do from demo-ceph01
sudo apt install -y uuid
FSID=$(uuid)

mkdir -p /etc/ceph
cat<<EOF | sudo tee /etc/ceph/ceph.conf
[global]
fsid = ${FSID}
mon initial members = demo-ceph01, demo-ceph02, demo-ceph03
mon host = 192.168.10.11, 192.168.10.12, 192.168.10.13
public network = 192.168.10.0/24
auth cluster required = cephx
auth service required = cephx
auth client required = cephx
EOF

# Distribute ceph.conf
scp /etc/ceph/ceph.conf demo-ceph02:/tmp/ceph.conf
ssh demo-ceph02 sudo -S mv /tmp/ceph.conf /etc/ceph/ceph.conf
scp /etc/ceph/ceph.conf demo-ceph03:/tmp/ceph.conf
ssh demo-ceph03 sudo -S mv /tmp/ceph.conf /etc/ceph/ceph.conf
```

2. Create ceph-mon keyring
```bash
sudo ceph-authtool \
  --create-keyring /tmp/ceph.mon.keyring \
  --gen-key -n mon. --cap mon 'allow *'
```

3. Create client.admin keyring
```bash
sudo ceph-authtool \
  --create-keyring /etc/ceph/ceph.client.admin.keyring \
  --gen-key -n client.admin \
  --cap mon 'allow *' \
  --cap osd 'allow *' \
  --cap mds 'allow *' \
  --cap mgr 'allow *'
```

4. Create bootstrap-osd keyring 
```bash
sudo ceph-authtool \
  --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring \
  --gen-key -n client.bootstrap-osd \
  --cap mon 'profile bootstrap-osd' \
  --cap mgr 'allow r'
```

5. Add keyring to ceph directory
```bash
sudo ceph-authtool /tmp/ceph.mon.keyring \
  --import-keyring /etc/ceph/ceph.client.admin.keyring
sudo ceph-authtool /tmp/ceph.mon.keyring \
  --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
```

6. Change ceph.mon.keyring ownership
```bash
sudo chown ceph:ceph /tmp/ceph.mon.keyring
```

7. Create ceph monitor map
```bash
echo $FSID
monmaptool --create \
  --add demo-ceph1 192.168.10.11 \
  --fsid ${FSID} /tmp/monmap

monmaptool \
  --add demo-ceph2 192.168.10.12 \
  --fsid ${FSID} /tmp/monmap

monmaptool \
  --add demo-ceph3 192.168.10.13 \
  --fsid ${FSID} /tmp/monmap
```

8. Distributes Keyring
```bash
# Distribute admin keyring
sudo cp /etc/ceph/ceph.client.admin.keyring /tmp/ceph.client.admin.keyring
sudo chown $USER:$USER /tmp/{ceph.client.admin.keyring,ceph.mon.keyring,monmap}

# Distribute admin keyring, mon keyring, dan monmap
scp /tmp/{ceph.client.admin.keyring,ceph.mon.keyring,monmap} demo-ceph02:/tmp
scp /tmp/{ceph.client.admin.keyring,ceph.mon.keyring,monmap} demo-ceph03:/tmp

# Move admin keyring to /etc/ceph
ssh demo-ceph02 sudo -S mv /tmp/ceph.client.admin.keyring /etc/ceph/
ssh demo-ceph03 sudo -S mv /tmp/ceph.client.admin.keyring /etc/ceph/
```

## Execute on all hosts
### Setup Ceph Mon
1. Create ceph mon directory
```bash
sudo -u ceph mkdir /var/lib/ceph/mon/ceph-$(hostname)
```

2. Initiate ceph mon
```bash
sudo chown ceph:ceph /tmp/ceph.mon.keyring
sudo -u ceph ceph-mon --mkfs -i $(hostname) --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
```

3. Enable and start ceph-mon
```bash
sudo systemctl enable --now ceph-mon@$(hostname)
sudo systemctl status ceph-mon@$(hostname)
```

4. Enable msgr2 monitor module
```bash
# On demo-ceph01
sudo chown $USER:$USER /etc/ceph/{ceph.conf,ceph.client.admin.keyring}
ceph mon enable-msgr2
```

5. Check cluster status 
```bash
ceph -s
```

### Setup Ceph Mgr
1. Create ceph-mgr keyring
```bash
sudo mkdir -p /var/lib/ceph/mgr/ceph-$(hostname)
sudo ceph auth get-or-create mgr.$(hostname) mon 'allow profile mgr' osd 'allow *' mds 'allow *' \
  -o /var/lib/ceph/mgr/ceph-$(hostname)/keyring
```

2. Create ceph-mgr directory
```bash
sudo chown -R ceph:ceph /var/lib/ceph/mgr
```

3. Enable and start service
```bash
sudo systemctl enable --now ceph-mgr@$(hostname)
sudo systemctl status ceph-mgr@$(hostname)
```

4. Check cluster status 
```bash
ceph -s
```

### Setup Ceph OSD
1. Prepare boostrap osd keyring
```bash
# Eksekusi dari demo-ceph01
sudo cp /var/lib/ceph/bootstrap-osd/ceph.keyring /tmp/ceph.keyring
sudo chown $USER:$USER /tmp/ceph.keyring

scp /tmp/ceph.keyring demo-ceph02:/tmp/ceph.keyring
scp /tmp/ceph.keyring demo-ceph03:/tmp/ceph.keyring
ssh demo-ceph02 sudo -S cp /tmp/ceph.keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
ssh demo-ceph03 sudo -S cp /tmp/ceph.keyring /var/lib/ceph/bootstrap-osd/ceph.keyring

sudo chown ceph:ceph /var/lib/ceph/bootstrap-osd/ceph.keyring

ssh demo-ceph02 sudo chown ceph:ceph /var/lib/ceph/bootstrap-osd/ceph.keyring
ssh demo-ceph03 sudo chown ceph:ceph /var/lib/ceph/bootstrap-osd/ceph.keyring
```

2. Deploy Ceph OSD
```bash
# Repeat on different host
sudo ceph-volume lvm create --bluestore --data /dev/vdb
sudo ceph-volume lvm create --bluestore --data /dev/vdc
sudo ceph-volume lvm create --bluestore --data /dev/vdd
```

3. Verify
```bash
ceph -s
ceph osd tree
```

# Fix mons are allowing insecure global_id reclaim 
```bash
ceph config set mon auth_expose_insecure_global_id_reclaim false
ceph config set mon auth_allow_insecure_global_id_reclaim false
```