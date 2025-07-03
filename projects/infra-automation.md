# Infrastructure Automation Projects

## 🖥️ Proxmox Virtualization Infrastructure

### 4-Node High-Availability Cluster
**Scale**: Production environment supporting 100+ VMs and containers

#### Architecture
```
┌─────────────────────────────────────────────────────┐
│                 Proxmox Cluster                     │
├─────────────┬─────────────┬────────────────────────┤
│   Node 0-1  │   Node 2-3  │   Ceph Storage Pool    │
│  (Primary)  │ (Secondary) │  (Distributed Across)  │
├─────────────┴─────────────┴────────────────────────┤
│            Ceph Distributed Storage                 │
│              ZFS Mirror Arrays                      │
└─────────────────────────────────────────────────────┘
```

**Key Implementations**:
- Ceph distributed storage for VM high availability
- ZFS arrays with automated snapshots and replication
- VLAN segregation for management, storage, and VM traffic
- Automated VM migration during maintenance windows
- Custom monitoring with Prometheus and Grafana

**Command Examples**:
```bash
# Automated cluster health check
pvecm status | grep -E "Quorate|Nodes"

# ZFS snapshot automation
zfs snapshot -r rpool/data@$(date +%Y%m%d-%H%M%S)
zfs send -R rpool/data@latest | ssh backup-node zfs recv -F backup/data

# Live migration script between nodes
qm migrate 101 node2 --online --with-local-disks
```

### LXC Container Orchestration
Developed automated deployment framework for containerized services:

```bash
# One-command LXC deployment
./deploy-lxc.sh --name cabinet-flow \
  --cores 4 --memory 8192 --disk 100 \
  --ip 10.10.20.40 --bridge vmbr0 \
  --features "nesting=1,keyctl=1"

# PM2 process management inside LXCs
pm2 start ecosystem.config.js
pm2 save
pm2 startup
```

## 🔒 Network Security Infrastructure

### Sophos XG Firewall HA Configuration
**Environment**: Dual XG330 appliances in active/passive configuration

**Implementation Details**:
- Hardware HA with dedicated heartbeat interface
- Synchronized security policies and VPN configurations
- Automated failover testing with sub-second switchover
- Integration with Active Directory for user-based policies
- SSL inspection with custom certificate deployment

**Configuration Highlights**:
```xml
<!-- HA Configuration Extract -->
<HAConfiguration>
  <Mode>Active-Passive</Mode>
  <HeartbeatInterface>Port8</HeartbeatInterface>
  <MonitoredInterfaces>Port1,Port2,Port3</MonitoredInterfaces>
  <FailoverTime>500ms</FailoverTime>
  <PreemptMode>Enabled</PreemptMode>
</HAConfiguration>
```

## ☁️ Azure Infrastructure & Migrations

### Virtual Desktop Infrastructure (AVD) Migration
**Project**: Law firm infrastructure modernization (200+ users)

**Architecture Transformation**:
- **Before**: On-premises RDS farm with persistent VDI
- **After**: Azure Virtual Desktop with FSLogix profiles

**Key Achievements**:
- Reduced infrastructure costs by 40% through autoscaling
- Implemented "Start VM on Connect" for cost optimization
- Automated session host deployment with custom images
- FSLogix profile containers with Azure Files integration

**PowerShell Automation**:
```powershell
# AVD Host Pool Autoscaling Configuration
$scalingPlan = @{
    Name = "BusinessHours"
    TimeZone = "Eastern Standard Time"
    ScheduledDays = "Monday,Tuesday,Wednesday,Thursday,Friday"
    RampUpStartTime = "07:00"
    PeakStartTime = "09:00"
    RampDownStartTime = "17:00"
    OffPeakStartTime = "20:00"
    
    # Scaling thresholds
    RampUpCapacityThreshold = 80
    RampUpMinimumHosts = 2
    PeakLoadBalancing = "BreadthFirst"
    RampDownCapacityThreshold = 20
    RampDownForceLogoff = $false
    OffPeakDisconnectTimeout = 15
}

New-AzWvdScalingPlan @scalingPlan
```

### Intune Device Management
**Scope**: 500+ endpoints across multiple locations

**Implementations**:
- Zero-touch Windows Autopilot deployment
- Conditional Access policies with risk-based authentication
- Application deployment via Company Portal
- Compliance policies with automated remediation

## 💾 Storage Infrastructure

### ZFS Storage Architecture
**Environment**: Multi-petabyte storage arrays

**Configuration**:
```bash
# ZFS Pool Creation with Optimal Performance
zpool create -f -o ashift=12 \
  -O compression=lz4 \
  -O atime=off \
  -O xattr=sa \
  -O recordsize=1M \
  storage-pool \
  mirror /dev/disk/by-id/disk1 /dev/disk/by-id/disk2 \
  mirror /dev/disk/by-id/disk3 /dev/disk/by-id/disk4

# Automated scrub scheduling
echo "0 2 * * 0 root /sbin/zpool scrub storage-pool" >> /etc/crontab

# Performance monitoring
zpool iostat -v storage-pool 5
```

### LVM Thin Provisioning
Implemented dynamic storage allocation for VM environments:

```bash
# Thin pool creation
lvcreate -L 10T -T vg_data/thinpool

# Automated thin volume provisioning
for i in {1..50}; do
  lvcreate -V 500G -T vg_data/thinpool -n vm_disk_$i
done

# Monitoring script for thin pool usage
#!/bin/bash
THRESHOLD=80
USAGE=$(lvs --noheadings -o data_percent vg_data/thinpool | tr -d ' ')
if (( $(echo "$USAGE > $THRESHOLD" | bc -l) )); then
  echo "Warning: Thin pool usage at ${USAGE}%"
  # Trigger automated expansion
fi
```

## 🔧 Infrastructure as Code

### Terraform Deployments
Created reusable modules for consistent infrastructure:

```hcl
# Proxmox VM Module
module "production_vm" {
  source = "./modules/proxmox-vm"
  
  name        = "app-server-${count.index + 1}"
  target_node = "node${count.index % 4}"
  cores       = 4
  memory      = 8192
  
  network = {
    bridge = "vmbr0"
    vlan   = var.production_vlan
  }
  
  count = 3
}
```

### Ansible Automation
Configuration management across heterogeneous environments:

```yaml
# Dynamic inventory for Proxmox
- name: Configure newly deployed VMs
  hosts: "{{ groups['proxmox_vms'] }}"
  tasks:
    - name: Base system configuration
      include_role:
        name: common
      vars:
        ntp_servers: "{{ datacenter_ntp }}"
        dns_servers: "{{ datacenter_dns }}"
    
    - name: Security hardening
      include_role:
        name: cis_hardening
      when: vm_type == "production"
```

## 📊 Monitoring & Observability

### Integrated Monitoring Stack
- **Prometheus**: Metrics collection from all infrastructure
- **Grafana**: Custom dashboards for each service
- **Alertmanager**: Intelligent alert routing
- **Loki**: Log aggregation and analysis

### Custom Exporter Development
Created specialized exporters for proprietary systems:

```python
# Proxmox cluster health exporter
class ProxmoxExporter:
    def collect(self):
        cluster_status = self.get_cluster_status()
        
        yield GaugeMetricFamily(
            'proxmox_cluster_quorum',
            'Cluster quorum status',
            value=1 if cluster_status['quorate'] else 0
        )
        
        yield GaugeMetricFamily(
            'proxmox_node_count',
            'Number of nodes in cluster',
            value=cluster_status['nodes']
        )
```

## 🚀 Performance Optimizations

### Storage Performance Tuning
- Implemented ZFS ARC tuning for optimal memory usage
- Configured SSD caching with L2ARC and SLOG devices
- Achieved 10GB/s+ sequential read performance

### Network Optimization
- Configured LACP bonding for redundancy and throughput
- Implemented jumbo frames across storage network
- Tuned TCP parameters for high-latency links

### Virtualization Optimization
- CPU pinning for latency-sensitive workloads
- NUMA awareness for memory allocation
- GPU passthrough for specialized workloads

## 📈 Results & Impact

- **Uptime**: Achieved 99.95% availability across all services
- **Performance**: 3x improvement in application response times
- **Cost Savings**: 40% reduction through intelligent autoscaling
- **Automation**: 80% reduction in manual operational tasks
- **Security**: Zero security incidents in 24 months

---

*These projects demonstrate my ability to design, implement, and maintain enterprise-grade infrastructure while continuously innovating with automation and emerging technologies.*