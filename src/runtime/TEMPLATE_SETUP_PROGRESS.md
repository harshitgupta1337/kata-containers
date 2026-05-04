# Trying to init template for CLH

## Memory-backing file created, but cannot connect to vsock

I can see that a CLH process is being created by executing `ps -aux`.

`/home/azureuser/cloud-hypervisor/target/release/cloud-hypervisor --api-socket 89a5977a-e8c2-4588-b582-7f172bab2c65/clh-api.sock`

However, it's api-socket is a relative path. Not sure if the runtime can reach the socket. Furthermore, the runtime logs show the following: `timed out connecting to hybrid vsocket hvsock:/clh.sock`.

Both the sockets exist. In fact, I can even connect by using `ch-remote` to the `clh-api.sock`, and using `socat` to the `clh.sock` as well. But the `clh.sock` agent socket also lived in the same dif as the CLH API sock. Now the question is, why is the runtime referring to the vsock as `hvsock:/clh.sock`? Is there a path mismatch?

## Answer: Two Different Sockets for Different Purposes

The issue stems from **two different sockets serving different purposes**:

1. **API Socket (`clh-api.sock`)**: Used for Cloud Hypervisor's REST API management (what `ch-remote` connects to)
2. **VSock Socket (`clh.sock`)**: Used for agent communication via hybrid vsocket

### Source of `hvsock:/clh.sock` format:

The `hvsock:/clh.sock` path comes from:
- `clh.go:GenerateSocket()` → creates `HybridVSock{UdsPath: clh.sock, Port: 1024}` 
- `types/sandbox.go:HybridVSock.String()` → formats as `"hvsock://path:port"`
- `HybridVSockScheme = "hvsock"` (defined in types/sandbox.go:38)

### The Path Construction:

```go
// In clh.go:1464
func (clh *cloudHypervisor) GenerateSocket(id string) (interface{}, error) {
    udsPath, err := clh.vsockSocketPath(id)  // -> calls BuildSocketPath(..., "clh.sock")
    return types.HybridVSock{
        UdsPath: udsPath,    // Full path to clh.sock  
        Port:    uint32(vSockPort), // 1024
    }, nil
}

// types/sandbox.go:211
func (s *HybridVSock) String() string {
    return fmt.Sprintf("%s://%s:%d", HybridVSockScheme, s.UdsPath, s.Port)
    //                 "hvsock"     full_path_to_clh.sock  1024
}
```

Both socket paths should be absolute (via `BuildSocketPath(clh.config.VMStorePath, id, socketName)`). If you're seeing relative paths, check the working directory where CLH process is launched.

## Root Cause Found: VMStorePath is Empty for Template VMs

The missing directory path in `hvsock:/clh.sock` is **intentional but problematic**:

### Why the Path is Incomplete:

1. **Template VM Creation**: In `factory/factory_linux.go:81`, `resetHypervisorConfig()` deliberately sets `VMStorePath = ""` for templates
2. **Socket Path Generation**: When `GenerateSocket()` calls `BuildSocketPath(clh.config.VMStorePath, id, clhSocket)`:
   - `clh.config.VMStorePath` is empty string `""`
   - `id` is the VM ID 
   - `clhSocket` is `"clh.sock"`
   - Result: `BuildSocketPath("", id, "clh.sock")` → just `"clh.sock"` (no directory)

3. **Formatted Result**: `HybridVSock.String()` produces `hvsock:/clh.sock` instead of `hvsock://full/path/to/clh.sock`

### The Design Issue:

Template VMs are meant to be path-agnostic, but the agent communication still needs working socket paths. The template creation process clears storage paths but the agent socket generation still depends on them.

**This explains why the connection times out** - the runtime is looking for a socket at just "clh.sock" relative to its working directory, not at the actual full path where CLH created it.