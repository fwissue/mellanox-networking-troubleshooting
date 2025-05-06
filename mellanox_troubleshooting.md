# Troubleshooting Mellanox Networking Issues

## Hardware & Link Issues

- Check cables, connectors, and transceivers (use `mlnxlink` to inspect link status).
- Ensure NIC and switch firmware are up to date:
  ```bash
  mlxfwmanager
  ```
- Verify PCIe slot compatibility and bandwidth:
  ```bash
  lspci | grep Mellanox
  ```

## Driver & Software Issues

- Install Mellanox OFED (`MLNX_OFED`) for proper driver support.
- Check driver status:
  ```bash
  ibstat        # For InfiniBand
  ethtool       # For Ethernet
  ```
- Restart networking:
  ```bash
  systemctl restart openibd
  ```

## Performance & RDMA Issues

- Verify RDMA is enabled:
  ```bash
  ibv_devinfo
  ```
- Check queue pair (QP) status:
  ```bash
  ibv_rc_pingpong -d mlx5_0
  ```
- Enable RoCE for low-latency Ethernet:
  ```bash
  ethtool -K rx on tx on
  ```

## Switch & Fabric Troubleshooting

- Check fabric topology:
  ```bash
  ibnetdiscover
  ```
- Diagnose switch issues with:
  ```bash
  mlnx_diag
  ibdiagnet
  ```

---

## 1. Check Hardware and Physical Links

### Check Link Status

Run:
```bash
ibstat
```
Look for `"LinkUp"` status. If the link is down, check the physical connection and firmware compatibility.

Verify per-port link speed:
```bash
ibstatus
```
If a link is down, check cable compatibility and switch/NIC port speeds.

### Verify Firmware & Driver

Check Mellanox Driver (MLNX_OFED) Version:
```bash
ofed_info -s
```
Ensure it's the latest version from Mellanox/NVIDIA.

Check Firmware Version:
```bash
mlxconfig query
```

To update firmware:
```bash
mlxfwmanager --query
mlxfwmanager --burn <firmware_file>
```

---

## 2. Diagnose Fabric & Network Issues

### Check Network Topology

```bash
ibnetdiscover
```
This helps visualize the InfiniBand fabric and detect missing or misconfigured nodes.

### Check All Nodes in the Fabric

```bash
ibhosts
```
Lists all hosts in the network to verify if any nodes are missing.

### Check Route to Remote Node

```bash
ibtracert <src_GUID> <dst_GUID>
```
If the route is broken, check fabric connections and subnet manager (SM).

---

### Subnet Manager (SM) Issues

Ensure Subnet Manager is running:
```bash
systemctl status opensm
```

If not running, start it:
```bash
systemctl start opensm
```

Check the logs for errors:
```bash
journalctl -u opensm --no-pager | tail -50
```

> Only **one subnet manager** should be active per fabric.  
> If multiple are running, use **priority settings** to avoid conflicts.

---

## 3. Check RDMA & Performance Issues

### Check RDMA Devices

```bash
ibv_devinfo
```
Confirms that the RDMA device is detected and operational.

### Test RDMA Connectivity

On one node (server):
```bash
ibv_rc_pingpong
```

On another node (client):
```bash
ibv_rc_pingpong <server_IP>
```

> If the test fails, check firewall settings, SM status, and IB link state.

### Check Packet Errors & Performance

```bash
perfquery -C
```
Helps detect dropped packets or high latency.

---

## Debugging Link & Performance Issues

### Check Port Counters on HCA (Host Channel Adapter)

```bash
iblinkinfo -r
```
Look for symbol errors, link down counts, or excessive retries.

### Check Switch Port Counters

On an IB switch:
```bash
ibswitches
```
Find the GUID, then check:
```bash
perfquery -G <switch_GUID>
```

> Identify ports with excessive errors and consider replacing cables or updating firmware.

---

## Additional Tools for Deep Debugging

### Run InfiniBand Fabric Diagnostic

```bash
ibdiagnet -r
```
Scans the fabric for errors, bad cables, or misconfigurations.

### Reset the Fabric (As a Last Resort)

```bash
opensm --restart
```
This restarts the subnet manager and may resolve connectivity issues.