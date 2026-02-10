# Troubleshooting

- Monitoring K8s resources
- Troubleshooting flow
- Troubleshooting K8s application

## Monitoring K8s resources

Using `kubectl get` can be used to get any resources. The command `kubectl describe` is useful which can display the details under the event section. If metrics-server installed `kubectl top` command should be available to check the pod and node utilization. 

## Troubleshooting Flow

- Resources first created in K8s etcd database, this can be check using `kubectl describe` and `kubectl event`. 
- After the resources added into the database, the Pod application is started on the node where it is scheduled.
- Before it can be started, the Pod image needs to be fetched:
    - Use `sudo crictl images` to get a list
- Once the application started, use `kubectl logs` to read output of the application or use the application shell - `kubectl exec -it [POD NAME] -- sh`
- In case of the absent of `ps` command, cat the `/proc/[PID]/cmdline` and `/proc/[PID]/status` file to see what proceess running. Example as below:
```
# ls /proc 
1   33          bus       crypto     dynamic_debug  interrupts  kcore        kpagecount     mdstat   mtrr          schedstat  stat           thread-self  version_signature
29  47          cgroups   devices    execdomains    iomem       key-users    kpageflags     meminfo  net           scsi       swaps          timer_list   vmallocinfo
30  acpi        cmdline   diskstats  fb             ioports     keys         latency_stats  misc     pagetypeinfo  self       sys            tty          vmstat
31  bootconfig  consoles  dma        filesystems    irq         kmsg         loadavg        modules  partitions    slabinfo   sysrq-trigger  uptime       zoneinfo
32  buddyinfo   cpuinfo   driver     fs             kallsyms    kpagecgroup  locks          mounts   pressure      softirqs   sysvipc        version
# cat /proc/1/cmdline
nginx: master process nginx -g daemon off;
# cat /proc/29/cmdline
nginx: worker process# 
# cat /proc/30/cmdline
nginx: worker process# 
```

## Troubleshooting K8s application

### Troubleshooting Pod

1st step is to use `kubectl get pod` for generic overview of the Pod. Pod can be in any of these states:
- Pending - already configured in etcd and waiting to schedule on eligible node
- Running - it is healthy 
- Succeeded - Pod has done it work and there is no need to restart 
- Failed - one or more Pod or container have ended with an error code and will not be restarted
- Unknown - state could not be obtained, often related to network issue
- Completed - Pod has run to completion
- CrashLoopBackOff - the pod generated an error, but still the scheduler try to run it

If `kubectl get` doesn't provided enough infomation then proceed to use `kubectl describe` to get more info.