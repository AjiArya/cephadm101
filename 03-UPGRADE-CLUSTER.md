# Part 3: Upgrade Ceph Cluster with Cephadm
# Pre-requisites
- Ceph Cluster (cephadm managed)
- Healthy Cluster

# Notes
The automated upgrade process follows Ceph best practices. For example:
- The upgrade order starts with managers, monitors, then other daemons.
- Each daemon is restarted only after Ceph indicates that the cluster will remain available.

Source: Ceph Docs

# Guide
1. Check current version
```bash
ceph versions
```

2. Upgrade command
```bash
ceph orch upgrade start --ceph-version 17.2.6
```

3. Watch the upgrade process
```bash
ceph -W cephadm
```

4. Verify the upgraded version
```bash
ceph versions
```