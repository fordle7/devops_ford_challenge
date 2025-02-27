# Problem 3: Diagnose Me Doctor

## Comprehensive Troubleshooting Guide for NGINX Load Balancer Memory Exhaustion

The persistent 99% memory utilization on an Ubuntu 24.04 virtual machine hosting an NGINX load balancer presents critical risks to system stability and service availability. 

This report outlines a systematic approach to diagnosing the root cause, analyzing potential scenarios, and implementing recovery measures.

---

### Step 1: Initial System Verification and Process Analysis

**Confirming Memory Utilization Patterns**

- Begin by distinguishing between actual memory consumption and cached/buffered memory using `free -m` or `htop`.
- Linux aggressively caches disk data, which may falsely inflate memory usage metrics. If the `available` column in `free -m` shows sufficient free memory, the issue may relate to misreported cached memory.
- However, a genuine 99% utilization demands immediate investigation into process-level allocation.

**Check Which Processes Are Consuming Memory**

- Execute `top` or `htop`, sorting processes by memory usage (**Shift+M**). If NGINX worker processes dominate memory consumption, proceed to analyze their configuration and connection patterns.
- Unexpected processes (e.g., `tracker-miner-fs`, `xdg-desktop-portal-gnome`) may indicate background services with leaks, as observed in Ubuntu 24.04 desktop environments.
- On servers, however, such services are typically absent, narrowing the focus to NGINX or kernel-level issues.

---

### Step 2: NGINX Configuration and Operational Review

**Worker Process and Connection Allocation**

- NGINX memory usage scales with `worker_processes` and `worker_connections`. Misconfigured values—such as setting `worker_processes` higher than CPU cores or excessively large `worker_connections`—can over-allocate memory.
- For example, a configuration with `worker_processes=8` and `worker_connections=4096` reserves 8 * 4096 * 2 (for accept/read buffers) * ~16 KB ≈ 1 GB, excluding dynamic allocations [`1`](https://loadforge.com/guides/monitoring-and-tuning-nginx-for-better-load-balancing).

**Buffer and Timeout Settings**

- Oversized buffer directives (`client_body_buffer_size`, `proxy_buffer_size`) inflate per-connection memory. 
- For instance, `client_body_buffer_size 10M` allocates 10 MB per request, which becomes unsustainable under high concurrency. Similarly, prolonged `keepalive_timeout` values retain idle connections, preventing memory reuse.

**gRPC and Streaming Protocols**

- Long-lived gRPC streams, as reported in NGINX 1.25.4, exhibit linear memory growth due to unmanaged chain allocations in `ngx_chain_writer()`[`2`](https://trac.nginx.org/nginx/ticket/2614). 
- If the load balancer handles gRPC or WebSocket traffic, verify upstream response handling and session termination.

---

### Step 3: Memory Leak Detection and Log Analysis

**Monitoring Tools and Techniques**

- Use `vmstat 1` to track memory trends over time. A steady increase in `free` memory depletion (not stabilized by caches) suggests a leak. 
- For NGINX-specific leaks, attach `valgrind --tool=massif` to a worker process, reproducing the workload to identify allocation hotspots [`2`](https://trac.nginx.org/nginx/ticket/2614).

**System and Kernel Logs**

- Inspect `/var/log/syslog` for OOM killer activity (Out of memory: Killed process entries).
- The OOM killer terminates processes when the system exhausts both RAM and swap, prioritizing those with high memory footprints. Frequent OOM events indicate unmitigated leaks or insufficient memory provisioning [`3`](https://cloud.google.com/dataproc/docs/support/troubleshoot-oom-errors).

**NGINX Error Logs**
- Review `/var/log/nginx/error.log` for warnings like ***upstream timed out*** or ***too many open files***, which correlate with resource exhaustion.
- High `worker_rlimit_nofile` values may mask file descriptor leaks, indirectly increasing memory pressure.

---
### Potential Root Causes and Mitigation Strategies

**Scenario 1: NGINX Memory Leak in gRPC Handling**

- **Cause**: A confirmed bug in NGINX’s gRPC module (versions ≤1.25.4) causes unbounded memory growth during long-lived stream processing [`2`](https://trac.nginx.org/nginx/ticket/2614).
- **Impact**: Gradual memory exhaustion leads to OOM crashes, disrupting traffic routing and triggering upstream service outages.
- **Recovery**:

    1. **Short-term**: Restart NGINX workers periodically via systemctl reload nginx to clear accumulated allocations.
    2. **Mid-term**: Apply patches from NGINX’s development branch or downgrade to a stable version without gRPC support.
    3. **Long-term**: Migrate gRPC traffic to dedicated proxies (e.g., Envoy) with better stream management.

**Scenario 2: Misconfigured Buffer or Connection Limits**

- **Cause**: Overprovisioned client_max_body_size or proxy_buffers forces NGINX to allocate excessive per-request memory.
- **Impact**: Sudden traffic spikes exhaust preallocated buffers, causing request failures and increased latency.
- **Recovery**:
    1. **Tune Buffers**: Set proxy_buffer_size 4k and proxy_buffers 8 4k to limit per-connection overhead.
    2. **Limit Connections**: Reduce worker_connections to match observed concurrency (netstat -tn | grep :80 | wc -l).
    3. **Enable Caching**: Offload repetitive requests via proxy_cache_path, reducing backend load and memory churn


**Scenario 3: System-Level Swap Misconfiguration**

- **Cause**: Insufficient or disabled swap space prevents the kernel from paging out inactive processes.
- **Impact**: The system cannot handle transient memory spikes, forcing abrupt OOM terminations.
- **Recovery**:
    1. **Expand Swap**: Use `swapspace` to dynamically manage swap files (`sudo apt install swapspace`).
    2. **Optimize Swappiness**: Set vm.swappiness=10 to prioritize RAM usage while allowing emergency paging.

**Scenario 4: Kernel or Driver Memory Leak**

- **Cause**: A rare kernel bug or hardware driver issue (e.g., AWS ENA driver) leaks memory outside user-space processes.
- **Impact**: Steady memory decline persists despite NGINX restarts, affecting all services on the VM.
- **Recovery**:
    1. **Update Kernel**: Upgrade to the latest HWE kernel (`sudo apt install linux-generic-hwe-24.04`).
    2. **Monitor Slab Memory**: Use `slabtop` to identify kernel objects (e.g., `dentry`, `inode_cache`) consuming disproportionate memory.

---
### Proactive Monitoring and Optimization

**Implementing Health Checks and Alerts**

- Configure Prometheus with the NGINX Exporter to track metrics like `nginx_connections_active` and `nginx_memory_usage`.
- Set alerts for sustained memory usage above 80% to enable preemptive scaling or tuning.

**Load Testing and Capacity Planning**

- Use `loadforge` or `wrk` to simulate traffic patterns, identifying thresholds where memory usage spikes.
- For example, a test revealing 1 GB/hour leakage under 1k RPS warrants horizontal scaling or architectural changes.

**NGINX Tuning Checklist**

1. **Worker Optimization:** Set `worker_processes auto`; and `worker_rlimit_nofile 65535;`.
2. **Connection Limits**: Define `worker_connections 2048;` within `events {}`.
3. **Buffer Adjustments**:
    ```
    client_body_buffer_size 16k;  
    client_header_buffer_size 1k;  
    proxy_buffers 8 16k;
    ```
4. **Timeout Reductions**:
    ```
    keepalive_timeout 30;  
    client_body_timeout 12;  
    client_header_timeout 12;  
    ```

---
### Conclusion and Recommendations

- The 99% memory utilization on the Ubuntu VM hosting NGINX likely stems from either a software leak (NGINX gRPC module), configuration oversights, or insufficient system resources. 
- Immediate steps include validating process memory footprints, adjusting NGINX buffer settings, and ensuring swap availability. Long-term stability requires upgrading NGINX, migrating stateful protocols to dedicated proxies, and implementing granular monitoring.
- By methodically isolating variables and applying targeted optimizations, system reliability can be restored while maintaining high-throughput load balancing.