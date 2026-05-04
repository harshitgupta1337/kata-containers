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

## New Challenge: Unimplemented VM State Management Functions

After adding the VMStorePath workaround, template creation now proceeds further but encounters another critical issue:

### Problem: Zero-Size Memory Files

Template creation completes but the `memory` file in the template directory (`/run/vc/vm/template/memory`) has **size 0**, which is unexpected. This indicates that the VM state is not being properly saved during template creation.

### Root Cause: Missing CLH Implementation

The issue stems from unimplemented functions in `virtcontainers/clh.go`:

- **`PauseVM()`**: Required to pause the VM before saving state
- **`SaveVM()`**: Required to persist the VM memory and state to disk

These functions are called during template creation in `template_linux.go`:

```go
func (t *template) createTemplateVM(ctx context.Context) error {
    // ... VM creation ...
    
    if err = vm.Pause(ctx); err != nil {  // -> calls unimplemented PauseVM()
        return err
    }

    if err = vm.Save(); err != nil {      // -> calls unimplemented SaveVM()
        return err
    }
    
    return nil
}
```

### Impact:

Without these implementations:
1. VM state cannot be properly saved to the template
2. Memory snapshots remain empty (0 bytes)
3. Template-based VM creation will fail or produce unusable VMs

### Next Steps:

The `PauseVM()` and `SaveVM()` functions in `virtcontainers/clh.go` need to be implemented to enable proper CLH template functionality. This likely involves:
- Using Cloud Hypervisor's snapshot/restore API
- Implementing proper state serialization
- Handling memory file persistence

## Problem 3: msg="fallback to direct factory vm" error="hypervisor config does not match"

When creating a sandbox from a template, this error is thrown in the logs. Due to the error, the template falls back to the `direct` mode of template creation.

The following snippet is where the config check fails: `virtcontainers/factory/factory_linux.go`.

```go
	err := f.checkConfig(config)
	if err != nil {
		f.log().WithError(err).Info("fallback to direct factory vm")
		return direct.New(ctx, config).GetBaseVM(ctx, config)
	}
```

### Root-cause: incomplete implementation of `resetHypervisorConfig` fn. in `virtcontainers/factory/factory_linux.go`

The following fields were not being reset, which caused a config mismatch error.

```
1. config.HypervisorConfig.SandboxName = ""
2. config.HypervisorConfig.SandboxNamespace = ""
// TODO: Check why DefaultMaxVCPUs needs to be reset.
3. config.HypervisorConfig.DefaultMaxVCPUs = 0
```

## Problem 4: load vm factory failed, about to create new one

This error is expected. CLH emits VM state as a `state.json` file.

```
error="stat /run/vc/vm/template/state: no such file or directory
```

Occuring here:

```go
func (t *template) checkTemplateVM() error {
	_, err := os.Stat(t.statePath + "/memory")
	if err != nil {
		return err
	}

	_, err = os.Stat(t.statePath + "/state")
	return err
}
```

## Problem 5: Apparently a brand new VM is being created because there is no call to restore

Sandbox creation (from the template) succeeds, BUT IT DOESN'T LOOK LIKE TEMPLATE WAS USED. This is my hypothesis because:
1. Time to create sandbox is almost as high as new one.
2. There is no explicit call to VM Restore in the Cloud-Hypervisor clh.go or anywhere else. There is only `ResumeVM` function, and that too has an empty implementation in CLH.


