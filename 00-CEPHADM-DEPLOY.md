# Pre-requisites
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

3. Install docker.io
```bash
sudo apt install -y docker.io
```

# Guide
1. (demo-ceph01) Install cephadm
```bash
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/quincy/src/cephadm/cephadm
chmod +x cephadm
sudo ./cephadm install
```

2. (demo-ceph01) Bootstrap ceph-mon
```bash
# With Monitoring Stack
sudo cephadm bootstrap --mon-ip 192.168.10.11 \
    --allow-fqdn-hostname \
    --no-minimize-config

# Skip Monitoring Stack
sudo cephadm bootstrap --mon-ip 192.168.10.11 \
    --allow-fqdn-hostname \
    --skip-monitoring-stack \
    --no-minimize-config
```

3. (All demo-ceph) Install ceph-common
```bash
sudo apt install -y ceph-common

```

4. Paste public key to all machine on the cluster
```bash
# Get ceph ssh public key from demo-ceph01
cat /etc/ceph/ceph.pub

# Add to root on all hosts (demo-ceph[01:03])
cat<<EOF | sudo tee -a /root/.ssh/authorized_keys
<CEPH_PUBLIC_KEY_CONTENT>
EOF
```

5. (demo-ceph01) Add hosts to cephadm and give labels
```bash
# Add machines
sudo ceph orch host add demo-ceph02 192.168.10.12 _admin
sudo ceph orch host add demo-ceph03 192.168.10.13 _admin

# Add Labels
sudo ceph orch host label add demo-ceph01 _admin
sudo ceph orch host label add demo-ceph01 mon
sudo ceph orch host label add demo-ceph02 mon
sudo ceph orch host label add demo-ceph03 mon
sudo ceph orch host label add demo-ceph01 mgr
sudo ceph orch host label add demo-ceph02 mgr
sudo ceph orch host label add demo-ceph03 mgr
sudo ceph orch host label add demo-ceph01 osd
sudo ceph orch host label add demo-ceph02 osd
sudo ceph orch host label add demo-ceph03 osd

# Verify
sudo ceph orch host ls
```

6. (demo-ceph01) Modify ceph-mon and ceph-mgr placement
```bash
cat<<EOF > ceph-mon-mgr.yaml
service_type: mon
placement:
  label: "mon"
---
service_type: mgr
placement:
  label: "mgr"
EOF

sudo ceph orch apply -i ceph-mon-mgr.yaml
```

7. (demo-ceph01) Create ceph-osd
```bash
cat<<EOF > ceph-osd.yaml
service_type: osd
service_id: osd # service_id bisa diisi secara bebas
placement:
  label: "osd"
data_devices:
  paths:
    - /dev/vdb
    - /dev/vdc
    - /dev/vdd
EOF

sudo ceph orch apply -i ceph-osd.yaml
```

8. (demo-ceph01) Verify cluster status
```bash
sudo ceph -s
```

# Fix mons are allowing insecure global_id reclaim 
```bash
sudo ceph config set mon auth_expose_insecure_global_id_reclaim false
sudo ceph config set mon auth_allow_insecure_global_id_reclaim false
```
