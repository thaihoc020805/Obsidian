## Why Limit Resources?

Without limits, a container can use:

- All available CPU cores
- All available memory
- All available disk I/O

This can cause problems like:

- One container slowing down others
- Memory leaks taking down the host
- High disk I/O impacting system performance

Think about running multiple containers on a single host - if one container starts consuming too many resources, it could impact all other containers. This is especially problematic in production environments where multiple services need to coexist reliably.

## Key Resource Limits

Let's look at the most important limits you can set:

### Memory Limits

Memory limits are probably the most important ones to set. A memory leak in one container could bring down your entire host if left unchecked!  

```
# Limit container to 512MB of memory
docker run -m 512m hello-world-go

# Same thing with memory swap disabled
docker run -m 512m --memory-swap 512m hello-world-go
```

The `--memory-swap` flag controls how much swap space the container can use. If set equal to `-m`, it prevents the container from using swap. This is often desirable because:

- Swap usage can severely impact performance
- It's better to have a container crash than slowly degrade system performance (Depends on what you are running of course)
- It makes memory issues more visible and easier to debug

When a container hits its memory limit, it will be killed by the OOM (Out Of Memory) killer.

### CPU Limits

CPU limits are trickier because they affect performance rather than reliability. There are two main approaches:  

```
# Limit container to use 50% of one CPU core
docker run --cpus=.5 hello-world-go

# Give container lower CPU priority (default is 1024)
docker run --cpu-shares=512 hello-world-go
```

The difference is important:

- `--cpus` is a hard limit - the container will never use more than this amount
- `--cpu-shares` is relative - it only matters when there's CPU contention
    - With `cpu-shares=512`, the container gets half as much CPU as others when busy
    - But if no other containers need CPU, it can use more

### Disk I/O Limits

I/O limits prevent one container from monopolizing disk access:  

```
# Limit container to 1MB per second write speed
docker run --device-write-bps /dev/sda:1mb hello-world-go

# Limit container to 100 IOPS for reads
docker run --device-read-iops /dev/sda:100 hello-world-go
```

This is particularly important for:

- Database containers that might do heavy I/O
- Batch processing jobs that read/write large files
- Preventing disk I/O from affecting latency-sensitive services

Of course, you need to know your actual device. `/dev/sda` is just an example!

## Real World Example

Remember our Go HTTP server from [Day 5](https://adventofdocker.com/posts/day-5-interacting-with-containers/)? Let's run it with resource limits:  

```
docker run -d \
  -p 8080:8080 \
  --name limited-server \
  --memory=256m \
  --cpus=0.5 \
  hello-world-go
```

This container:

- Can only use 256MB of RAM
- Uses at most 50% of one CPU core

You can verify the limits are working:  

```
docker stats limited-server
CONTAINER ID   NAME             CPU %     MEM USAGE / LIMIT     MEM %
abc123def456   limited-server   0.00%     2.5MiB / 256MiB      0.98%
```

The stats command gives you real-time feedback on resource usage. This is invaluable for:

- Debugging performance issues
- Tuning resource limits
- Monitoring container health