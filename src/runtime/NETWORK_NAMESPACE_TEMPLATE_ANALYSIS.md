# Network Namespace Handling in Kata Templates

This document describes how network namespaces are handled in Kata's VM templating system, specifically focusing on the absence of network configuration in template VMs and the dynamic attachment of network namespaces when creating sandboxes from templates.

## Overview

Kata's template system deliberately separates VM state (CPU, memory, hypervisor configuration) from network configuration. This design ensures network isolation between container instances while maximizing template reusability and performance benefits.

## Key Architectural Principle

**Template VMs contain NO network namespace configuration** - they are created in a network-agnostic state to ensure:
- Network isolation between containers
- Template reusability across different network configurations
- Security boundary maintenance
- Flexibility in network policy application

## Template VM Creation - Network Absence

### 1. Template VM Creation Process

**File**: `virtcontainers/factory/template/template_linux.go` (lines 115-148)

```go
func (t *template) createTemplateVM(ctx context.Context) error {
    // create the template vm
    config := t.config
    config.HypervisorConfig.BootToBeTemplate = true
    config.HypervisorConfig.BootFromTemplate = false
    config.HypervisorConfig.MemoryPath = t.statePath + "/memory"
    config.HypervisorConfig.DevicesStatePath = t.statePath + "/state"

    vm, err := vc.NewVM(ctx, config)  // No network config passed
    if err != nil {
        return err
    }
    // ... template creation continues
}
```

**Key Point**: No network configuration is passed to the template VM creation process.

### 2. Empty Network Creation

**File**: `virtcontainers/vm.go` (line 93)

```go
network, err := NewNetwork()  // Called with no arguments
```

This creates an empty network configuration:

**File**: `virtcontainers/network_linux.go` (lines 62-70)

```go
func NewNetwork(configs ...*NetworkConfig) (Network, error) {
    // Empty constructor - no network namespace
    if len(configs) == 0 {
        return &LinuxNetwork{}, nil  // netNSPath is empty string
    }
    // ...
}
```

**Result**: Template VM has `LinuxNetwork` with:
- `netNSPath = ""` (empty string)
- `netNSCreated = false`
- `eps = []` (no endpoints)

### 3. Hypervisor Configuration

During template creation, the hypervisor (e.g., Cloud Hypervisor) is configured with:
- Memory zones for sharing
- CPU configuration
- Device state paths
- **NO network devices or interfaces**

## Sandbox Creation from Template - Network Attachment

### 1. Sandbox Network Configuration Entry Point

**File**: `virtcontainers/sandbox.go` (line 673)

```go
network, err := NewNetwork(&sandboxConfig.NetworkConfig)
```

**Critical Difference**: Unlike template creation, sandbox creation receives `sandboxConfig.NetworkConfig` containing:
- `NetworkID`: Path to existing network namespace (e.g., `/proc/12345/ns/net`)
- `NetworkCreated`: Boolean indicating if namespace was created by runtime
- `InterworkingModel`: Network interworking model to use

### 2. Network Namespace Path Association

**File**: `virtcontainers/network_linux.go` (lines 75-82)

```go
return &LinuxNetwork{
    netNSPath:         config.NetworkID,  // Points to existing network namespace
    eps:               []Endpoint{},
    interworkingModel: config.InterworkingModel,
    netNSCreated:      config.NetworkCreated,
    danConfigPath:     config.DanConfigPath,
}, nil
```

**Key Point**: `NetworkID` contains the path to a network namespace that **already exists**, created by the container runtime (containerd, CRI-O, etc.) and configured by CNI plugins.

### 3. Network Interface Discovery Process

**File**: `virtcontainers/network_linux.go` (line 341)

```go
func (n *LinuxNetwork) addAllEndpoints(ctx context.Context, s *Sandbox, hotplug bool) error {
    endpoints, err := n.scanEndpointsInNs(ctx, s, n.netNSPath, hotplug)
    // ...
}
```

### 4. Network Namespace Scanning and Interface Detection

**File**: `virtcontainers/network_linux.go` (lines 394-457)

```go
func (n *LinuxNetwork) scanEndpointsInNs(ctx context.Context, s *Sandbox, nsPath string, hotplug bool) ([]Endpoint, error) {
    // Open the network namespace
    netnsHandle, err := netns.GetFromPath(nsPath)
    if err != nil {
        return nil, err
    }
    defer netnsHandle.Close()

    // Create netlink handle for the namespace
    netlinkHandle, err := netlink.NewHandleAt(netnsHandle)
    if err != nil {
        return nil, err
    }
    defer netlinkHandle.Close()

    // Discover all network interfaces in the namespace
    linkList, err := netlinkHandle.LinkList()
    if err != nil {
        return nil, err
    }

    // Process each discovered interface
    for _, link := range linkList {
        netInfo, err := networkInfoFromLink(netlinkHandle, link)
        if err != nil {
            return nil, err
        }

        // Skip unconfigured interfaces and loopback
        if len(netInfo.Addrs) == 0 || (netInfo.Iface.Flags & net.FlagLoopback) != 0 {
            continue
        }

        // Create endpoint and attach to VM
        if err := doNetNS(nsPath, func(_ ns.NetNS) error {
            ep, addErr := n.addSingleEndpoint(ctx, s, netInfo, hotplug)
            if addErr == nil {
                added = append(added, ep)
            }
            return addErr
        }); err != nil {
            return nil, err
        }
    }

    return added, nil
}
```

### 5. Network Namespace Context Switching

**File**: `virtcontainers/network_linux.go` (lines 1328-1358)

```go
func doNetNS(netNSPath string, cb func(ns.NetNS) error) error {
    // Template VMs have empty netNSPath, so callback runs in current namespace
    if netNSPath == "" {
        var netNs ns.NetNS
        return cb(netNs)
    }

    // For sandbox VMs, switch to target network namespace
    runtime.LockOSThread()
    defer runtime.UnlockOSThread()

    currentNS, err := ns.GetCurrentNS()
    if err != nil {
        return err
    }
    defer currentNS.Close()

    targetNS, err := ns.GetNS(netNSPath)  // Get handle to target namespace
    if err != nil {
        return err
    }

    if err := targetNS.Set(); err != nil {  // Switch to target namespace
        return err
    }
    defer currentNS.Set()  // Switch back to original namespace

    return cb(targetNS)  // Execute network operations in target namespace
}
```

## Network Namespace Lifecycle

### Template VM Lifecycle
1. **Creation**: No network namespace attached (`netNSPath = ""`)
2. **Boot**: VM boots without network interfaces
3. **Pause**: VM paused in network-agnostic state
4. **Storage**: Template saved with shared memory but no network state

### Sandbox VM Lifecycle
1. **Template Load**: VM memory and CPU state loaded from template
2. **Network Discovery**: Container runtime provides network namespace path
3. **Interface Scanning**: Kata discovers interfaces in the provided namespace
4. **Endpoint Creation**: Network endpoints created for each discovered interface
5. **Hypervisor Attachment**: Network devices hot-plugged to running VM
6. **Agent Configuration**: Guest agent informed of network configuration

## Network Interface Types Supported

The network discovery process supports multiple interface types:

```go
// From addSingleEndpoint function
switch {
case isPhysical:
    endpoint, err = createPhysicalEndpoint(netInfo)
case socketPath != "":
    endpoint, err = createVhostUserEndpoint(netInfo, socketPath)
case netInfo.Iface.Type == "macvlan":
    endpoint, err = createMacvlanNetworkEndpoint(idx, netInfo.Iface.Name, n.interworkingModel)
case netInfo.Iface.Type == "macvtap":
    endpoint, err = createMacvtapNetworkEndpoint(netInfo)
case netInfo.Iface.Type == "tap":
    endpoint, err = createTapNetworkEndpoint(idx, netInfo.Iface.Name)
case netInfo.Iface.Type == "veth":
    endpoint, err = createVethNetworkEndpoint(idx, netInfo.Iface.Name, n.interworkingModel)
case netInfo.Iface.Type == "ipvlan":
    endpoint, err = createIPVlanNetworkEndpoint(idx, netInfo.Iface.Name)
}
```

## Hypervisor Network Device Attachment

### Hot-plug Process

When network endpoints are discovered, they are hot-plugged to the running VM:

```go
// From addSingleEndpoint function
if hotplug {
    if err := endpoint.HotAttach(ctx, s); err != nil {
        return nil, err
    }
} else {
    if err := endpoint.Attach(ctx, s); err != nil {
        return nil, err
    }
}
```

### Cloud Hypervisor Integration

For Cloud Hypervisor, network devices are added via `vmAddNetPut()` API calls after VM boot, enabling dynamic network configuration without affecting the base template.

## Security and Isolation Benefits

### 1. Network Isolation
- Each container gets its own network namespace
- No network state shared between template and instances
- Container runtime controls network policy

### 2. Template Security
- Templates cannot leak network information
- No network-based attack vectors through template
- Clean separation of compute and network resources

### 3. Flexibility
- Same template works with different network configurations
- CNI plugins can configure networking independently
- Network policies applied per-container, not per-template

## Performance Implications

### 1. Template Benefits Preserved
- Fast VM boot from shared memory state
- No network initialization overhead for template
- Memory sharing maximized

### 2. Dynamic Network Cost
- Network discovery adds minimal overhead
- Hot-plug operations are fast
- Interface detection is cached

### 3. Optimization Opportunities
- Network interface pre-warming possible
- Endpoint caching across container lifecycle
- Network device templates (future enhancement)

## Error Handling and Fallbacks

### 1. Network Namespace Detection
```go
// From detectHypervisorNetns function
func (n *LinuxNetwork) detectHypervisorNetns(s *Sandbox) (string, bool) {
    pid, err := s.GetHypervisorPid()
    if err != nil || pid <= 0 {
        return "", false
    }
    
    hypervisorNs := fmt.Sprintf("/proc/%d/ns/net", pid)
    // Compare namespace inodes to detect different namespaces
    // ...
}
```

### 2. Namespace Fallback
If the originally specified namespace has no interfaces, Kata can detect and switch to the hypervisor's network namespace (useful for Docker 26+ scenarios).

### 3. Cleanup on Failure
- Failed network operations trigger endpoint cleanup
- Namespace switching is atomic with proper rollback
- Resource cleanup prevents leaks

## Configuration Examples

### Template Configuration
```toml
# Template factory enabled
[factory]
template = true
template_path = "/var/lib/kata/template"

# No network configuration needed for template
```

### Runtime Network Configuration
```json
{
  "NetworkConfig": {
    "NetworkID": "/proc/12345/ns/net",
    "NetworkCreated": false,
    "InterworkingModel": "macvtap"
  }
}
```

## Summary

The Kata template system's approach to network namespace handling demonstrates a sophisticated separation of concerns:

1. **Templates are network-agnostic**: Contain only VM compute state
2. **Sandboxes are network-aware**: Dynamically discover and attach to existing network namespaces
3. **Runtime integration**: Container runtimes control network policy and configuration
4. **Security preserved**: Network isolation maintained between containers
5. **Performance optimized**: Template benefits retained while enabling flexible networking

This architecture ensures that Kata's VM templating provides maximum performance benefits while maintaining the security and flexibility expected from container runtimes.