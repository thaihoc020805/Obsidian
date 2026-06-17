## Docker ps - Container Status

The `docker ps` command is your first stop for debugging. It shows running containers and their basic information:  

```
$ docker ps
CONTAINER ID   IMAGE            COMMAND   CREATED          STATUS          PORTS                    NAMES
abc123def456   hello-world-go   "./main"  20 minutes ago   Up 20 minutes   0.0.0.0:8080->8080/tcp   myapp
```

Some useful flags:

- `-a`: Show all containers (including stopped ones)
- `-q`: Only display container IDs (useful for scripting)
- `-f status=exited`: Filter containers by status

## Docker Logs - Container Output

When your container isn't behaving as expected, checking its logs is crucial:  

```
$ docker logs myapp
Listening on port 8080
```

Useful flags:

- `-f`: Follow log output (like `tail -f`)
- `--tail 100`: Show last 100 lines
- `--since 5m`: Show logs from last 5 minutes

For our Go server, we can watch incoming requests in real-time:  

```
$ docker logs -f myapp
Listening on port 8080
New visit recorded at 2024-12-14T10:00:00Z
New visit recorded at 2024-12-14T10:01:00Z
```

## Docker Inspect - Deep Dive

`docker inspect` provides detailed information about a container:  

``` json
$ docker inspect myapp
[
    {
        "Id": "f915b41c43f7cadb2be97e30b6d7eac9ec0f04beed3ffd1d47b62d7c4f6d649a",
        "Created": "2024-12-14T18:07:45.903730046Z",
        "Path": "./main",
        "Args": [],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 1564,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2024-12-14T18:07:46.19197188Z",
            "FinishedAt": "0001-01-01T00:00:00Z"
        },
        "Image": "sha256:6209fff2e82261192905db51004792b13e688c44729e457299daa5230b2f8c31",
        "ResolvConfPath": "/var/lib/docker/containers/f915b41c43f7cadb2be97e30b6d7eac9ec0f04beed3ffd1d47b62d7c4f6d649a/resolv.conf",
        "HostnamePath": "/var/lib/docker/containers/f915b41c43f7cadb2be97e30b6d7eac9ec0f04beed3ffd1d47b62d7c4f6d649a/hostname",
        "HostsPath": "/var/lib/docker/containers/f915b41c43f7cadb2be97e30b6d7eac9ec0f04beed3ffd1d47b62d7c4f6d649a/hosts",
        "LogPath": "/var/lib/docker/containers/f915b41c43f7cadb2be97e30b6d7eac9ec0f04beed3ffd1d47b62d7c4f6d649a/f915b41c43f7cadb2be97e30b6d7eac9ec0f04beed3ffd1d47b62d7c4f6d649a-json.log",
        "Name": "/relaxed_mayer",
        "RestartCount": 0,
        "Driver": "overlay2",
        "Platform": "linux",
        "MountLabel": "",
        "ProcessLabel": "",
        "AppArmorProfile": "",
        "ExecIDs": null,
        "HostConfig": {
            "Binds": null,
            "ContainerIDFile": "",
            "LogConfig": {
                "Type": "json-file",
                "Config": {}
            },
            "NetworkMode": "default",
            "PortBindings": {},
            "RestartPolicy": {
                "Name": "no",
                "MaximumRetryCount": 0
            },
            "AutoRemove": false,
            "VolumeDriver": "",
            "VolumesFrom": null,
            "ConsoleSize": [
                22,
                97
            ],
            "CapAdd": null,
            "CapDrop": null,
            "CgroupnsMode": "private",
            "Dns": [],
            "DnsOptions": [],
            "DnsSearch": [],
            "ExtraHosts": null,
            "GroupAdd": null,
            "IpcMode": "private",
            "Cgroup": "",
            "Links": null,
            "OomScoreAdj": 0,
            "PidMode": "",
            "Privileged": false,
            "PublishAllPorts": false,
            "ReadonlyRootfs": false,
            "SecurityOpt": null,
            "UTSMode": "",
            "UsernsMode": "",
            "ShmSize": 67108864,
            "Runtime": "runc",
            "Isolation": "",
            "CpuShares": 0,
            "Memory": 0,
            "NanoCpus": 0,
            "CgroupParent": "",
            "BlkioWeight": 0,
            "BlkioWeightDevice": [],
            "BlkioDeviceReadBps": [],
            "BlkioDeviceWriteBps": [],
            "BlkioDeviceReadIOps": [],
            "BlkioDeviceWriteIOps": [],
            "CpuPeriod": 0,
            "CpuQuota": 0,
            "CpuRealtimePeriod": 0,
            "CpuRealtimeRuntime": 0,
            "CpusetCpus": "",
            "CpusetMems": "",
            "Devices": [],
            "DeviceCgroupRules": null,
            "DeviceRequests": null,
            "MemoryReservation": 0,
            "MemorySwap": 0,
            "MemorySwappiness": null,
            "OomKillDisable": null,
            "PidsLimit": null,
            "Ulimits": [],
            "CpuCount": 0,
            "CpuPercent": 0,
            "IOMaximumIOps": 0,
            "IOMaximumBandwidth": 0,
            "MaskedPaths": [
                "/proc/asound",
                "/proc/acpi",
                "/proc/kcore",
                "/proc/keys",
                "/proc/latency_stats",
                "/proc/timer_list",
                "/proc/timer_stats",
                "/proc/sched_debug",
                "/proc/scsi",
                "/sys/firmware"
            ],
            "ReadonlyPaths": [
                "/proc/bus",
                "/proc/fs",
                "/proc/irq",
                "/proc/sys",
                "/proc/sysrq-trigger"
            ]
        },
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/c8b77f1281e2a923debc57ba8ee8a3e61b2eab01c3746707847c56aa504f4549-init/diff:/var/lib/docker/overlay2/plebc717q0dae0vpogboi82uj/diff:/var/lib/docker/overlay2/0vwf87fyclw8wmvgnjngza8ul/diff:/var/lib/docker/overlay2/02a45bf063940c5ed96d43abf699c49e729c0197a8922613a7b1e85454397fd9/diff:/var/lib/docker/overlay2/1f7886b749ec4d3a50ff5a88e082404c777b05cb8af10469eb8d32f283eaa2fd/diff:/var/lib/docker/overlay2/813de0000b1ac87c0ebbc260ab5db42a860e771927966b98a08a6eb942a7478e/diff:/var/lib/docker/overlay2/bb1ea71019466bdd2da7dccf480c104121842b694d6022f96367214f6f623557/diff:/var/lib/docker/overlay2/aa54f03b2bcd787f34371e483e4b14024920790f54d8bdf2dda70c828022d3d0/diff:/var/lib/docker/overlay2/3caedf6970d13d30fd04cb8553c19a1dc1012d1474474f9e38dd992eb5a08e08/diff:/var/lib/docker/overlay2/c2ff2ee7edd0d1efda1493a24508b0bb8132464f0ea3a600572bb823a6f43708/diff",
                "MergedDir": "/var/lib/docker/overlay2/c8b77f1281e2a923debc57ba8ee8a3e61b2eab01c3746707847c56aa504f4549/merged",
                "UpperDir": "/var/lib/docker/overlay2/c8b77f1281e2a923debc57ba8ee8a3e61b2eab01c3746707847c56aa504f4549/diff",
                "WorkDir": "/var/lib/docker/overlay2/c8b77f1281e2a923debc57ba8ee8a3e61b2eab01c3746707847c56aa504f4549/work"
            },
            "Name": "overlay2"
        },
        "Mounts": [],
        "Config": {
            "Hostname": "f915b41c43f7",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": true,
            "AttachStderr": true,
            "ExposedPorts": {
                "8080/tcp": {}
            },
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/go/bin:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "GOLANG_VERSION=1.23.4",
                "GOTOOLCHAIN=local",
                "GOPATH=/go"
            ],
            "Cmd": [
                "./main"
            ],
            "Image": "hello-world-go",
            "Volumes": null,
            "WorkingDir": "/go",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": {}
        },
        "NetworkSettings": {
            "Bridge": "",
            "SandboxID": "f790e6709650efe5af98f6caf40b365b541b8005f4da2a4a636297c26e2a13d4",
            "HairpinMode": false,
            "LinkLocalIPv6Address": "",
            "LinkLocalIPv6PrefixLen": 0,
            "Ports": {
                "8080/tcp": null
            },
            "SandboxKey": "/var/run/docker/netns/f790e6709650",
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "01cb1873a9821ed1369e79274537ba2234676130a295cbfc5def7a8e0059101a",
            "Gateway": "172.17.0.1",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "IPAddress": "172.17.0.2",
            "IPPrefixLen": 16,
            "IPv6Gateway": "",
            "MacAddress": "02:42:ac:11:00:02",
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "ee1f25dbee96b37d4acbdbaee780065e892468e515104b6e7c590be39365c3e5",
                    "EndpointID": "01cb1873a9821ed1369e79274537ba2234676130a295cbfc5def7a8e0059101a",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.2",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:02",
                    "DriverOpts": null
                }
            }
        }
    }
]
```

Now this might be a bit overwhelming, but it's a good starting point to understand what's going on with your container! Once you know what exactly you're looking for, you can use Go templates to extract specific information:  

```
# Get IP address
$ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' myapp

# Get environment variables
$ docker inspect -f '{{range .Config.Env}}{{println .}}{{end}}' myapp
```

## Container Dumps - Advanced Debugging

For more serious issues, you can create a container dump:  

```
$ docker container export myapp > container.tar
```

This creates a tarball containing the container's entire filesystem, which you can then examine:  

```
$ tar -tvf container.tar | grep main
-rwxr-xr-x 0/0      1839488 2024-12-14 10:00 ./main
```

## Resource Usage and Cleanup

Sometimes your apps might run out of disk space. You can check how much space Docker is using with `docker system df`:  

```
$ docker system df
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          5         3         1.92GB    1.2GB
Containers      3         1         125MB     125MB
Local Volumes   2         2         256MB     0B
```

### Clean Up Resources

When debugging leads to many unused containers and images, clean up with:  

```
$ docker system prune
WARNING! This will remove:
  - all stopped containers
  - all networks not used by at least one container
  - all dangling images
  - all dangling build cache

Are you sure you want to continue? [y/N]
```

Add `-a` to remove all unused images, not just dangling ones:  

```
$ docker system prune -a
```

## Real-World Example

Let's debug a common issue with our Go server. Imagine it's not responding on port 8080:

1. Check if container is running:

```
$ docker ps
CONTAINER ID   IMAGE            STATUS          PORTS
# No container shown - it's not running
```

1. Check stopped containers:

```
$ docker ps -a
CONTAINER ID   IMAGE            STATUS                     PORTS
def456abc789   hello-world-go   Exited (1) 2 minutes ago
```

1. Check logs for error:

```
$ docker logs def456abc789
bind: address already in use
```

1. Inspect container for port bindings:

```
$ docker inspect def456abc789 -f '{{json .HostConfig.PortBindings}}'
{"8080/tcp":[{"HostIp":"","HostPort":"8080"}]}
```

Now we know the issue: port 8080 is already in use on the host. We can fix this by using a different port:  

```
$ docker run -p 8081:8080 hello-world-go
```