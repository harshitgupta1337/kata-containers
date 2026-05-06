# Kata Factory Control Flow: Template VM Creation and Management

This document provides a comprehensive overview of the control flow for the `kata-ctl factory init` command and the sandbox templating mechanism in Kata Containers.

## Overview

The Kata factory system enables VM templating, which provides significant performance improvements by pre-booting a VM and sharing its memory state across multiple container instances. This eliminates the cold start overhead for subsequent containers.

## Control Flow Architecture

### 1. CLI Entry Point

**File**: `cmd/kata-runtime/factory.go`  
**Function**: `initFactoryCommand.Action` (lines 149-213)

This is the main CLI command handler invoked when `kata-ctl factory init` is executed.

#### Key Operations:
- Validates runtime configuration
- Creates factory configuration with template settings
- Determines whether to create VM cache or template
- Calls the appropriate factory constructor

```go
var initFactoryCommand = cli.Command{
    Name:  "init",
    Usage: "initialize a VM factory based on kata-runtime configuration",
    Action: func(c *cli.Context) error {
        // Configuration validation and factory creation
        if runtimeConfig.FactoryConfig.Template {
            _, err := vf.NewFactory(ctx, factoryConfig, false)
            // ...
        }
    },
}
```

### 2. Factory Creation Dispatcher

**File**: `virtcontainers/factory/factory_linux.go`  
**Function**: `NewFactory()` (line 28-68)

Acts as a dispatcher that determines the appropriate factory type based on configuration.

#### Factory Types:
- **Template Factory**: When `config.Template = true`
- **Cache Factory**: When `config.Cache > 0`
- **Direct Factory**: Default fallback
- **gRPC Cache**: For VM cache client scenarios

```go
func NewFactory(ctx context.Context, config Config, fetchOnly bool) (vc.Factory, error) {
    if config.Template {
        if fetchOnly {
            b, err = template.Fetch(config.VMConfig, config.TemplatePath)
        } else {
            b, err = template.New(ctx, config.VMConfig, config.TemplatePath)
        }
    }
    // ... other factory types
}
```

### 3. Template Factory Initialization

**File**: `virtcontainers/factory/template/template_linux.go`  
**Function**: `New()` (line 43-68)

Creates a new VM template factory with the following sequence:

#### Initialization Steps:
1. **Validation**: Checks if template already exists
2. **File Preparation**: Sets up tmpfs and memory files via `prepareTemplateFiles()`
3. **Template Creation**: Creates the template VM via `createTemplateVM()`
4. **Cleanup Setup**: Defers cleanup on failure

```go
func New(ctx context.Context, config vc.VMConfig, templatePath string) (base.FactoryBase, error) {
    t := &template{templatePath, config}
    
    err := t.checkTemplateVM()
    if err == nil {
        return nil, fmt.Errorf("There is already a VM template in %s", templatePath)
    }
    
    err = t.prepareTemplateFiles()
    if err != nil {
        return nil, err
    }
    
    err = t.createTemplateVM(ctx)
    if err != nil {
        return nil, err
    }
    
    return t, nil
}
```

### 4. Template VM Creation Core Logic

**File**: `virtcontainers/factory/template/template_linux.go`  
**Function**: `createTemplateVM()` (line 115-148)

This is where the actual template VM is created and configured.

#### Template VM Creation Steps:
1. **Template Configuration**: Sets `BootToBeTemplate = true`
2. **Path Configuration**: Configures memory and device state paths
3. **VM Creation**: Calls `vc.NewVM()` to create the VM
4. **VM Management**: Disconnects, sleeps for agent cleanup, and pauses VM

```go
func (t *template) createTemplateVM(ctx context.Context) error {
    config := t.config
    config.HypervisorConfig.BootToBeTemplate = true
    config.HypervisorConfig.BootFromTemplate = false
    config.HypervisorConfig.MemoryPath = t.statePath + "/memory"
    config.HypervisorConfig.DevicesStatePath = t.statePath + "/state"
    
    vm, err := vc.NewVM(ctx, config)
    if err != nil {
        return err
    }
    defer vm.Stop(ctx)
    
    if err = vm.Disconnect(ctx); err != nil {
        return err
    }
    
    // Sleep to let agent cleanup and restart
    time.Sleep(templateWaitForAgent)
    
    if err = vm.Pause(ctx); err != nil {
        return err
    }
    
    return vm.Save(ctx)
}
```

### 5. VM Creation Orchestration

**File**: `virtcontainers/vm.go`  
**Function**: `NewVM()` (line 86-168)

Orchestrates the complete VM creation process with hypervisor integration.

#### VM Creation Steps:
1. **Hypervisor Setup**: Creates hypervisor instance via `NewHypervisor()`
2. **Network Setup**: Initializes network configuration
3. **VM Configuration**: Calls `hypervisor.CreateVM()` for hypervisor-specific setup
4. **Agent Setup**: Configures and validates guest agent
5. **VM Boot**: Calls `hypervisor.StartVM()` to start the VM
6. **Agent Verification**: Checks agent connectivity (skipped for templates)

```go
func NewVM(ctx context.Context, config VMConfig) (*VM, error) {
    // 1. setup hypervisor
    hypervisor, err := NewHypervisor(config.HypervisorType)
    if err != nil {
        return nil, err
    }
    
    // 2. Create VM
    if err = hypervisor.CreateVM(ctx, id, network, &config.HypervisorConfig); err != nil {
        return nil, err
    }
    
    // 3. setup agent
    agent := newAagentFunc()
    err = agent.configure(ctx, hypervisor, id, vmSharePath, config.AgentConfig)
    
    // 4. boot up guest vm
    if err = hypervisor.StartVM(ctx, VmStartTimeout); err != nil {
        return nil, err
    }
    
    // 5. Check agent aliveness (skipped for templates)
    if !config.HypervisorConfig.BootFromTemplate {
        err = agent.check(ctx)
    }
    
    return &VM{...}, nil
}
```

### 6. Cloud Hypervisor Integration

**File**: `virtcontainers/clh.go`

The Cloud Hypervisor implementation provides the low-level VM management:

#### CreateVM Function (line 749-869):
- Sets up internal VM configuration
- Configures memory zones for templating
- Prepares hypervisor-specific settings

#### StartVM Function (line 1017+):
- Launches the Cloud Hypervisor process
- Boots the VM via `bootVM()`
- Sets up VirtioFS daemon if needed

```go
// Memory configuration for templating
memoryZoneConfig := chclient.NewMemoryZoneConfig("mem0", int64((utils.MemUnit(clh.config.MemorySize) * utils.MiB).ToBytes()))
memoryZoneConfig.SetShared(true)
memoryZoneConfig.SetFile("/dev/shm/app-snapshot/template.mem")
clh.vmconfig.Memory.Zones = &[]chclient.MemoryZoneConfig{
    *memoryZoneConfig,
}
```

## Key Configuration Flags

### Template Creation Flags
- **`BootToBeTemplate = true`**: Indicates this VM will become a template
- **`BootFromTemplate = false`**: Used during template creation
- **`MemoryPath`**: Points to shared memory file for the template
- **`DevicesStatePath`**: Points to device state file for the template

### Template Usage Flags
- **`BootFromTemplate = true`**: Used when creating VMs from template
- **`BootToBeTemplate = false`**: Used for regular VM instances

## Template Storage and File Management

### File Preparation (`prepareTemplateFiles()`)

**Location**: `virtcontainers/factory/template/template_linux.go` (line 95+)

#### Operations:
1. **Directory Creation**: Creates template state directory
2. **Tmpfs Mount**: Mounts tmpfs for shared memory
3. **Memory File**: Creates memory file for VM state sharing

```go
func (t *template) prepareTemplateFiles() error {
    // Create directory
    err := os.MkdirAll(t.statePath, 0700)
    
    // Mount tmpfs with size calculation
    flags := uintptr(syscall.MS_NOSUID | syscall.MS_NODEV)
    opts := fmt.Sprintf("size=%dM", t.config.HypervisorConfig.MemorySize+templateDeviceStateSize)
    if err = syscall.Mount("tmpfs", t.statePath, "tmpfs", flags, opts); err != nil {
        return err
    }
    
    // Create memory file
    f, err := os.Create(t.statePath + "/memory")
    if err != nil {
        return err
    }
    f.Close()
    
    return nil
}
```

## Template Usage Flow

### Creating VMs from Templates

When creating new VMs from an existing template:

1. **Template Fetch**: `template.Fetch()` loads existing template
2. **VM Cloning**: `createFromTemplateVM()` creates new VM from template
3. **Configuration**: Sets `BootFromTemplate = true`
4. **Memory Sharing**: Uses same memory and device state paths
5. **VM Resumption**: Template VM gets resumed and cloned

### Performance Benefits

The templating mechanism provides:
- **Faster Boot Times**: Eliminates cold start overhead
- **Memory Sharing**: Shared memory pages between VM instances
- **Resource Efficiency**: Reduced memory footprint for multiple containers
- **Consistent State**: All VMs start from the same known-good state

## Factory Types and Use Cases

### Template Factory
- **Use Case**: Pre-boot VMs for faster container startup
- **Configuration**: `Template = true`
- **Storage**: Persistent template files on disk

### Cache Factory
- **Use Case**: Keep pre-booted VMs in memory cache
- **Configuration**: `Cache > 0`
- **Storage**: In-memory VM instances

### gRPC Cache Factory
- **Use Case**: Remote VM cache via gRPC
- **Configuration**: `VMCache = true, Cache = 0`
- **Storage**: Remote cache server

### Direct Factory
- **Use Case**: No optimization, create VMs on-demand
- **Configuration**: Default fallback
- **Storage**: No persistent state

## Error Handling and Cleanup

### Template Creation Failures
- Automatic cleanup via deferred functions
- Tmpfs unmounting and directory removal
- Process termination and resource cleanup

### Template Usage Failures
- VM state restoration from template
- Fallback to direct VM creation
- Resource cleanup and error propagation

## Configuration Integration

The factory system integrates with Kata's configuration system through:

- **Runtime Config**: `oci.RuntimeConfig.FactoryConfig`
- **Hypervisor Config**: VM-specific settings
- **Agent Config**: Guest agent configuration
- **Template Path**: Storage location for template files

## Future Considerations

- **Multi-architecture Support**: Template compatibility across architectures
- **Storage Optimization**: Template compression and deduplication
- **Network Templates**: Pre-configured network states
- **Security Isolation**: Template integrity and validation
- **Monitoring**: Template health and performance metrics

## Summary

The Kata factory system provides a sophisticated VM templating mechanism that significantly improves container startup performance. The control flow spans from CLI command handling through multiple abstraction layers down to hypervisor-specific implementation, with robust error handling and resource management throughout the stack.