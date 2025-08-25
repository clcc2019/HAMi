# Real-time GPU Status Check

## Overview

HAMi now supports real-time GPU status checking during scheduling to prevent scheduling conflicts when non-HAMi managed pods occupy GPU memory. This feature addresses the issue where HAMi scheduler cannot detect GPU memory usage by pods not managed by HAMi, leading to over-allocation and out-of-memory errors.

## Features

- **Real-time GPU Memory Detection**: Query actual GPU memory usage using NVML
- **Configurable Check Modes**: Support strict, warning, and disabled modes
- **Per-Pod Control**: Enable/disable real-time checking per pod via annotations
- **Safety Margins**: Built-in safety margins to prevent edge cases
- **Fallback Support**: Graceful fallback to cached data when real-time check fails

## Configuration

### Global Configuration

Enable real-time checking globally by setting the scheduler flag:

```bash
# Enable real-time checking for all pods (unless overridden by pod annotation)
--enable-realtime-check=true
```

### Environment Variables

For NVIDIA device plugin, set the following environment variables:

```bash
# Enable real-time checking for NVIDIA devices
HAMI_ENABLE_REALTIME_CHECK=true

# Set real-time check mode (strict|warning|disabled)
HAMI_REALTIME_CHECK_MODE=strict
```

### Per-Pod Configuration

Control real-time checking per pod using annotations:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-gpu-pod
  annotations:
    # Enable real-time checking for this pod
    hami.io/enable-realtime-check: "true"
spec:
  containers:
  - name: gpu-container
    image: nvidia/cuda:11.0-base
    resources:
      limits:
        nvidia.com/gpu: 1
        nvidia.com/gpumem: 4096
```

## Check Modes

### Strict Mode (`HAMI_REALTIME_CHECK_MODE=strict`) - Default

- **Behavior**: Reject scheduling if real-time memory check fails
- **Use Case**: Production environments where strict memory isolation is required
- **Risk**: May reject valid scheduling requests due to temporary memory spikes

### Warning Mode (`HAMI_REALTIME_CHECK_MODE=warning`)

- **Behavior**: Log warnings but allow scheduling to proceed
- **Use Case**: Most production environments, provides visibility without blocking
- **Risk**: May still lead to OOM in extreme cases

### Disabled Mode (`HAMI_REALTIME_CHECK_MODE=disabled`)

- **Behavior**: Skip real-time memory validation
- **Use Case**: Fallback mode or when real-time checking is not needed
- **Risk**: Same as original behavior without real-time checking

## Implementation Details

### NVIDIA GPU Support

Real-time checking is currently implemented for NVIDIA GPUs using NVML library:

- **Memory Usage**: Queries actual GPU memory usage via `nvmlDeviceGetMemoryInfo`
- **Utilization**: Retrieves GPU utilization via `nvmlDeviceGetUtilizationRates`
- **Process Count**: Counts running processes via `nvmlDeviceGetComputeRunningProcesses`
- **Safety Margin**: Applies 100MB safety margin to prevent edge cases

### Other Device Types

For non-NVIDIA devices, the interface returns "not implemented" errors:
- Hygon DCU devices
- Cambricon MLU devices
- Iluvatar GPU devices
- Mthreads GPU devices
- Enflame GCU devices
- Ascend NPU devices
- Kunlun XPU devices
- AWS Neuron devices
- Metax GPU/SGPU devices

Future implementations can be added by implementing the `GetRealTimeDeviceUsage` method.

## Scheduling Flow

The enhanced scheduling flow with real-time checking:

1. **Pod Submission**: User submits pod with GPU resource requests
2. **Webhook Interception**: HAMi webhook intercepts and modifies pod
3. **Scheduler Invocation**: Kubernetes calls HAMi scheduler extender
4. **Node Filtering**: For each candidate node:
   - Get cached device usage from annotations
   - **[NEW]** If real-time check enabled: Query actual GPU status via NVML
   - **[NEW]** Validate real-time memory availability
   - Apply existing filters (type, UUID, cores, etc.)
5. **Device Allocation**: Allocate devices on selected node
6. **Pod Binding**: Bind pod to selected node

## Monitoring and Debugging

### Log Messages

Real-time checking generates detailed log messages:

```bash
# Successful real-time check
I0101 12:00:00.000000 scheduler.go:564] Updated real-time device usage nodeID="node-1" deviceID="GPU-12345" cachedUsedMemMB=0 realTimeUsedMemMB=2048 utilization=45 processCount=1

# Failed real-time check (fallback to cached data)
I0101 12:00:00.000000 scheduler.go:551] Failed to get real-time usage, using cached data nodeID="node-1" deviceID="GPU-12345" error="NVML not initialized"

# Real-time memory check failure
I0101 12:00:00.000000 device.go:710] Real-time memory check failed pod="default/my-pod" device="GPU-12345" requestedMem=4096 realUsedMem=6144 totalMem=8192
```

### Metrics

Monitor real-time checking effectiveness:
- Check success/failure rates
- Memory usage accuracy
- Scheduling decision changes

## Troubleshooting

### Common Issues

1. **NVML Initialization Failures**
   ```
   Error: failed to initialize NVML: NVML_ERROR_DRIVER_NOT_LOADED
   ```
   - **Solution**: Ensure NVIDIA drivers are properly installed
   - **Workaround**: Disable real-time checking

2. **Permission Errors**
   ```
   Error: failed to get device handle: NVML_ERROR_NO_PERMISSION
   ```
   - **Solution**: Run scheduler with appropriate permissions
   - **Alternative**: Use warning mode instead of strict mode

3. **Device Not Found**
   ```
   Error: failed to get device handle for UUID GPU-12345: NVML_ERROR_NOT_FOUND
   ```
   - **Solution**: Verify device UUID consistency between device plugin and scheduler
   - **Check**: Ensure device plugin is running and registering devices correctly

### Performance Considerations

- **Latency**: Real-time checks add ~1-5ms per device per scheduling decision
- **CPU Usage**: Minimal CPU overhead for NVML queries
- **Memory**: No significant memory overhead
- **Scalability**: Tested with up to 8 GPUs per node

## Migration Guide

### Upgrading from Previous Versions

1. **Update HAMi Components**:
   ```bash
   # Update scheduler with new flags
   kubectl patch deployment hami-scheduler -p '{"spec":{"template":{"spec":{"containers":[{"name":"scheduler","args":["--enable-realtime-check=true"]}]}}}}'
   ```

2. **Update Device Plugin**:
   ```bash
   # Set environment variables
   kubectl patch daemonset hami-device-plugin -p '{"spec":{"template":{"spec":{"containers":[{"name":"device-plugin","env":[{"name":"HAMI_ENABLE_REALTIME_CHECK","value":"true"}]}]}}}}'
   ```

3. **Gradual Rollout**:
   - Start with warning mode
   - Monitor logs and metrics
   - Switch to strict mode if needed

### Rollback Procedure

If issues occur, disable real-time checking:

```bash
# Disable globally
kubectl patch deployment hami-scheduler -p '{"spec":{"template":{"spec":{"containers":[{"name":"scheduler","args":["--enable-realtime-check=false"]}]}}}}'

# Or set environment variable
kubectl patch daemonset hami-device-plugin -p '{"spec":{"template":{"spec":{"containers":[{"name":"device-plugin","env":[{"name":"HAMI_ENABLE_REALTIME_CHECK","value":"false"}]}]}}}}'
```

## Future Enhancements

- **Multi-GPU Topology**: Consider GPU interconnect topology in real-time checks
- **Historical Data**: Track GPU usage patterns for better predictions
- **Other Vendors**: Implement real-time checking for AMD, Intel, and other GPU vendors
- **Custom Metrics**: Export real-time GPU metrics to Prometheus
- **Predictive Scheduling**: Use ML models to predict GPU memory usage

## Contributing

To add real-time checking support for new device types:

1. Implement the `GetRealTimeDeviceUsage` method in your device struct
2. Set `IsRealTimeCheckEnabled()` to return `true`
3. Add appropriate vendor-specific libraries for device querying
4. Test with various workload scenarios
5. Update documentation

See the NVIDIA implementation in `pkg/device/nvidia/device.go` for reference.
