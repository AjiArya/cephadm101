# Part 2: Convert Ceph to Cephadm Managed
# Pre-requisites
- Setup Manual Ceph Cluster (Daemon)

# Guide
1. Use root user
```bash
sudo -i
```

2. Install docker & cephadm on all hosts
```bash
# Install docker
apt install -y docker.io 

# Install cephadm
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
chmod +x cephadm
./cephadm install

# verifikasi
dpkg -l cephadm

# Prepare
cephadm prepare-host
ceph config assimilate-conf -i /etc/ceph/ceph.conf

# Verify config assimilate
ceph config dump
```

3. Migrate ceph-mon
```bash
# Repeat on different host after ceph-mon successfully run as container
cephadm adopt --style legacy --name mon.$(hostname)
```

4. Migrate ceph-mgr
```bash
# Repeat on different host after ceph-mgr successfully run as container
cephadm adopt --style legacy --name mgr.$(hostname)
```

5. Enable cephadm mgr module
```bash
# Execute only once on any host
ceph mgr module enable cephadm
ceph orch set backend cephadm
```

6. Generate cephadm SSH key
```bash
# Execute only once on any host
ceph cephadm generate-key
ceph cephadm get-pub-key > ~/ceph.pub
```

7. Register SSH key
```bash
# Copy content of SSH public key
cat ~/ceph.pub

# Distribute to all hosts
## Make sure to copy SSH Public key as root user
sudo -i
cat<<EOF >> ~/.ssh/authorized_keys
<CEPHADM_SSH_PUBLIC_KEY>
EOF
```

8. Register all hosts to cephadm
```bash
ceph orch host add demo-ceph01 192.168.10.11 _admin
ceph orch host add demo-ceph02 192.168.10.12 _admin
ceph orch host add demo-ceph03 192.168.10.13 _admin

ceph orch host label add demo-ceph01 mon
ceph orch host label add demo-ceph02 mon
ceph orch host label add demo-ceph03 mon

ceph orch host label add demo-ceph01 mgr
ceph orch host label add demo-ceph02 mgr
ceph orch host label add demo-ceph03 mgr

ceph orch host label add demo-ceph01 osd
ceph orch host label add demo-ceph02 osd
ceph orch host label add demo-ceph03 osd

# Verify
ceph orch ls

# Apply ceph components placement
ceph orch apply mon --placement=label:mon
ceph orch apply mgr --placement=label:mgr
```

9. Migrate ceph-osd
```bash
# Stop ceph-crash
systemctl stop ceph-crash.service
systemctl stop ceph-osd.target

# Change ceph uid & gid
usermod -u 167 ceph && groupmod -g 167 ceph

# Get the osd id
ceph-volume lvm list | grep === | awk '{print $2}'

# Migrate OSD gradually
cephadm adopt --style legacy --name <osd_id>

# Convert all at once
# for osd_id in $(ceph-volume lvm list | grep === | awk '{print $2}'); do
#   echo cephadm adopt --style legacy --name $osd_id
#   cephadm adopt --style legacy --name $osd_id
# done

# Start if no OSD started
# systemctl start ceph-{cluster_id}@osd.{osd_id}.service
#
# or start all OSD
for osd_id in $(ceph-volume lvm list | grep === | awk '{print $2}'); do
  echo systemctl start ceph-$(grep fsid /etc/ceph/ceph.conf | awk '{print $3}')@$osd_id
  systemctl start ceph-$(grep fsid /etc/ceph/ceph.conf | awk '{print $3}')@$osd_id
done
```

# Workaround
## Unable to start / Error Ceph Process (mon, mgr, osd)
### Solution
1. Change ceph user id to 167
```bash
usermod -u 167 ceph && groupmod -g 167 ceph
```

2. (If previous command unsuccess) Disable ceph-mgr
```bash
systemctl stop ceph-crash
```

3. Retry
```bash
usermod -u 167 ceph && groupmod -g 167 ceph
```

4. Redeploy process
```bash
# Check error process
ceph orch ps

# Redeploy (required authentication to ceph cluster)
ceph orch redeploy <process_name>

# All at once (required authentication to ceph cluster)
# for i in $(ceph orch ps | grep $(hostname) | grep error | awk '{print $1}'); do ceph orch redeploy $i; done
```
