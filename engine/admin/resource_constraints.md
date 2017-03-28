---
redirect_from:
  - "/engine/articles/systemd/"
title: "Limit a container's resources"
description: "Limiting the system resources a container can use"
keywords: "docker, daemon, configuration"
---

By default, a container has no resource constraints and can use as much of a
given resource as the host's kernel scheduler will allow. Docker provides ways
to control how much memory, CPU, or block IO a container can use, setting runtime
configuration flags of the `docker run` command. This section provides details
on when you should set such limits and the possible implications of setting them.

## Memory

Docker can enforce hard memory limits, which allow the container to use no more
than a given amount of user or system memory, or soft limits, which allow the
container to use as much memory as it needs unless certain conditions are met,
such as when the kernel detects low memory or contention on the host machine.
Some of these options have different effects when used alone or when more than
one option is set.

Most of these options take a positive integer, followed by a suffix of `b`, `k`,
`m`, `g`, to indicate bytes, kilobytes, megabytes, or gigabytes.

| Option                | Description                 |
|-----------------------|-----------------------------|
| `-m` or `--memory=` | The maximum amount of memory the container can use. If you set this option, the minimum allowed value is `4m` (4 megabyte). |
| `--memory-swap`*    | The amount of memory this container is allowed to swap to disk. See [`--memory-swap` details](resource_constraints.md#memory-swap-details). |
| `--memory-swappiness` | By default, the host kernel can swap out a percentage of anonymous pages used by a container. You can set `--memory-swappiness` to a value between 0 and 100, to tune this percentage. See [`--memory-swappiness` details](resource_constraints.md#memory-swappiness-details). |
| `--memory-reservation` | Allows you to specify a soft limit smaller than `--memory` which is activated when Docker detects contention or low memory on the host machine. If you use `--memory-reservation`, it must be set lower than `--memory` in order for it to take precedence. Because it is a soft limit, it does not guarantee that the container will not exceed the limit. |
| `--kernel-memory` | The maximum amount of kernel memory the container can use. The minimum allowed value is `4m`. Because kernel memory cannot be swapped out, a container which is starved of kernel memory may block host machine resources, which can have side effects on the host machine and on other containers. See [`--kernel-memory` details](resource_constraints.md#kernel-memory-details). |
| `--oom-kill-disable` | By default, if an out-of-memory (OOM) error occurs, the kernel kills processes in a container. To change this behavior, use the `--oom-kill-disable` option. Only disable the OOM killer on containers where you have also set the `-m/--memory` option. If the `-m` flag is not set, the host can run out of memory and the kernel may need to kill the host system's processes to free memory. |

For more information about cgroups and memory in general, see the documentation
for [Memory Resource Controller](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt).

### `--memory-swap` details

- If unset, and `--memory` is set, the container can use twice as much swap
      as the `--memory` setting, if the host container has swap memory configured.
      For instance, if `--memory="300m"` and `--memory-swap` is not set, the
      container can use 300m of memory and 600m of swap.
- If set to a positive integer,  and if both `--memory` and `--memory-swap`
      are set, `--memory-swap` represents the total amount of memory and swap
      that can be used, and `--memory` controls the amount used by non-swap
      memory. So if `--memory="300m"` and `--memory-swap="1g"`, the container
      can use 300m of memory and 700m (1g - 300m) swap.
- If set to `-1` (the default), the container is allowed to use unlimited swap memory.

### `--memory-swappiness` details

- A value of 0 turns off anonymous page swapping.
- A value of 100 sets all anonymous pages as swappable.
- By default, if you do not set `--memory-swappiness`, the value is
  inherited from the host machine.

### `--kernel-memory` details

Kernel memory limits are expressed in terms of the overall memory allocated to
a container. Consider the following scenarios:

- **Unlimited memory, unlimited kernel memory**: This is the default
  behavior.
- **Unlimited memory, limited kernel memory**: This is appropriate when the
  amount of memory needed by all cgroups is greater than the amount of
  memory that actually exists on the host machine. You can configure the
  kernel memory to never go over what is available on the host machine,
  and containers which need more memory need to wait for it.
- **Limited memory, umlimited kernel memory**: The overall memory is
  limited, but the kernel memory is not.
- **Limited memory, limited kernel memory**: Limiting both user and kernel
  memory can be useful for debugging memory-related problems. If a container
  is using an unexpected amount of either type of memory, it will run out
  of memory without affecting other containers or the host machine. Within
  this setting, if the kernel memory limit is lower than the user memory
  limit, running out of kernel memory will cause the container to experience
  an OOM error. If the kernel memory limit is higher than the user memory
  limit, the kernel limit will not cause the container to experience an OOM.

When you turn on any kernel memory limits, the host machine tracks "high water
mark" statistics on a per-process basis, so you can track which processes (in
this case, containers) are using excess memory. This can be seen per process
by viewing `/proc/<PID>/status` on the host machine.

## CPU

By default, each container's access to the host machine's CPU cycles is unlimited.
You can set various constraints to limit a given container's access to the host
machine's CPU cycles. Most users will use and configure the
[default CFS scheduler](#configure-the-default-cfs-scheduler). In Docker 1.13
and higher, you can also configure the
[realtime scheduler](#configure-the-realtime-scheduler).

### Configure the default CFS scheduler

The CFS is the Linux kernel CPU scheduler for normal Linux processes. Several
runtime flags allow you to configure the amount of access to CPU resources your
container has. When you use these settings, Docker modifies the settings for 
the container's cgroup on the host machine.

| Option                | Description                 |
|-----------------------|-----------------------------|
| `--cpus=<value>`      | Specify how much of the available CPU resources a container can use. For instance, if the host machine has two CPUs and you set `--cpus="1.5"`, the container will be guaranteed to be able to access at most one and a half of the CPUs. This is the equivalent of setting `--cpu-period="100000"` and `--cpu-quota="150000"`. Available in Docker 1.13 and higher. |
| `--cpu-period=<value>`| Specify the CPU CFS scheduler period, which is used alongside  `--cpu-quota`. Defaults to 1 second, expressed in micro-seconds. Most users do not change this from the default. If you use Docker 1.13 or higher, use `--cpus` instead. |
| `--cpu-quota=<value>` | Impose a CPU CFS quota on the container. The number of microseconds per `--cpu-period` that the container is guaranteed CPU access. In other words, `cpu-quota / cpu-period`. If you use Docker 1.13 or higher, use `--cpus` instead. |
| `--cpuset-cpus`       | Limit the specific CPUs or cores a container can use. A comma-separated list or hyphen-separated range of CPUs a container can use, if you have more than one CPU. The first CPU is numbered 0. A valid value might be `0-3` (to use the first, second, third, and fourth CPU) or `1,3` (to use the second and fourth CPU). |
| `--cpu-shares`        | Set this flag to a value greater or less than the default of 1024 to increase or reduce the container's weight, and give it access to a greater or lesser proportion of the host machine's CPU cycles. This is only enforced when CPU cycles are constrained. When plenty of CPU cycles are available, all containers use as much CPU as they need. In that way, this is a soft limit. `--cpu-shares` does not prevent containers from being scheduled in swarm mode. It prioritizes container CPU resources for the available CPU cycles. It does not guarantee or reserve any specific CPU access. |

If you have 1 CPU, each of the following commands will guarantee the container at
most 50% of the CPU every second.

**Docker 1.13 and higher**:

```bash
docker run -it --cpus=".5" ubuntu /bin/bash
```

**Docker 1.12 and lower**:

```bash
$ docker run -it --cpu-period=100000 --cpu-quota=50000 ubuntu /bin/bash
```

### Configure the realtime scheduler

In Docker 1.13 and higher, you can configure your container to use the
realtime scheduler, for tasks which cannot use the CFS scheduler. You need to
[make sure the host machine's kernel is configured correctly](#configure-the-host-machines-kernel)
before you can [configure the Docker daemon](#configure-the-docker-daemon) or
[configure individual containers](#configure-individual-containers).

>**Warning**: CPU scheduling and prioritization are advanced kernel-level
features. Most users do not need to change these values from their defaults.
Setting these values incorrectly can cause your host system to become unstable
or unusable.

#### Configure the host machine's kernel

Verify that `CONFIG_RT_GROUP_SCHED` is enabled in the Linux kernel by running
`zcat /proc/config.gz | grep CONFIG_RT_GROUP_SCHED` or by checking for the
existence of the file `/sys/fs/cgroup/cpu.rt_runtime_us`. For guidance on
configuring the kernel realtime scheduler, consult the documentation for your
operating system.

#### Configure the Docker daemon

To run containers using the realtime scheduler, run the Docker daemon with
the `--cpu-rt-runtime` flag set to the maximum number of microseconds reserved
for realtime tasks per runtime period. For instance, with the default period of
10000 microseconds (1 second), setting `--cpu-rt-runtime=95000` ensures that
containers using the realtime scheduler can run for 95000 microseconds for every
10000-microsecond period, leaving at least 5000 microseconds available for
non-realtime tasks. To make this configuration permanent on systems which use
`systemd`, see [Control and configure Docker with systemd](systemd.md).

#### Configure individual containers

You can pass several flags to control a container's CPU priority when you
start the container using `docker run`. Consult your operating system's
documentation or the `ulimit` command for information on appropriate values.

| Option                    | Description                 |
|---------------------------|-----------------------------|
| `--cap-add=sys_nice`      | Grants the container the `CAP_SYS_NICE` capability, which allows the container to raise process `nice` values, set real-time scheduling policies, set CPU affinity, and other operations. |
| `--cpu-rt-runtime=<value>`| The maximum number of microseconds the container can run at realtime priority within the Docker daemon's realtime scheduler period. You also need the `--cap-add=sys_nice` flag. |
| `--ulimit rtprio=<value>` | The maximum realtime priority allowed for the container. You also need the `--cap-add=sys_nice` flag. |

The following example command sets each of these three flags on a `debian:jessie`
container.

```bash
$ docker run --it --cpu-rt-runtime=95000 \
                  --ulimit rtprio=99 \
                  --cap-add=sys_nice \
                  debian:jessie
```

If the kernel or Docker daemon is not configured correctly, an error will occur.