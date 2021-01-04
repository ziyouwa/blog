# Ubuntu20.10下安装podman

Unbuntu20.10自带的源已经完整支持podman，而且安装十分简单：

```
~$ apt install -y  podman
...
The following additional packages will be installed:
  buildah conmon containernetworking-plugins crun fuse-overlayfs fuse3
  golang-github-containers-common libavahi-glib1 libfuse3-3 libostree-1-1 libslirp0 libyajl2
  slirp4netns tini uidmap
Suggested packages:
  containers-storage
The following packages will be REMOVED:
  fuse
The following NEW packages will be installed:
  buildah conmon containernetworking-plugins crun fuse-overlayfs fuse3
  golang-github-containers-common libavahi-glib1 libfuse3-3 libostree-1-1 libslirp0 libyajl2
  podman slirp4netns tini uidmap
0 upgraded, 16 newly installed, 1 to remove and 0 not upgraded.
Need to get 27.4 MB of archives.
After this operation, 125 MB of additional disk space will be used.
Do you want to continue? [Y/n]

```

如上图所示，该命令会自动安装podman、crun、buildah和slirp4netns。但是，接下来有两个小小的问题需要解决。

## 没有运行时？

安装完毕后，直接执行podman info，会提示如下：

```
~$ podman info
Error: default OCI runtime "runc" not found: invalid argument
```

大概意思就是没有找到默认的容器运行时runc。

搜索的结果，大多数都是[建议直接安装runc](https://github.com/containers/podman/issues/6072)。但是如果您阅读的时候很仔细，就会发现上面安装podman时，已经安装了一个叫crun的东西。它到底是什么呢？和runc有什么关系？在[这里](https://www.redhat.com/sysadmin/introduction-crun)可以找到答案，大意就是二者都是容器运行时，只不过crun更小，更先进，文章作者建议是用crun。既然ubuntu20.10默认安装的是crun，那么我们就来研究一下，到底是什么原因导致podman没能识别它呢？

我们继续找，发现了[这个](https://serverfault.com/questions/989509/how-can-i-change-the-oci-runtime-in-podman)。经过验证，有两个配置文件都能设置podman的runtime，使用其中之一即可。推荐使用containers.conf，因为ubuntu20.10里面我并没有找到libpod.conf，不知道是不是旧版本的配置文件名。它们分别是

libpod.conf

```
runtime = "crun"
```

和containers.conf

```
[engine]
runtime = "crun"
```

/etc/container/目录下默认存在containers.conf文件，可以修改它，针对所有用户起作用，或是放在$HOME/.config/containers目录下，只针对当前用户生效。建议后者，毕竟podman的优势之一就是不需要root权限执行。

## 为什么CGROUP还是V1？



执行podman info命令，现在可以正常得到信息了，不过我们可以看到，下面的输出内容里面，写明了cgroupVersion还是v1。明明crun的优势就是支持v2啊。

```
:~$ podman info
....
host:
  arch: amd64
  buildahVersion: 1.15.2
  cgroupVersion: v1

```

通过mount命令，我们可以看到系统默认都挂载了v1和v2版本

```
~$ mount
...
cgroup2 on /sys/fs/cgroup/unified type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate)
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,name=systemd)
...
```

既然支持新版了，那么就去掉v1吧，具体步骤如下，参考在[[这里](https://mbien.dev/blog/tags/debian)]:

* 编辑/etc/default/grub

* 添加systemd.unified_cgroup_hierarchy=1到 GRUB_CMDLINE_LINUX_DEFAULT里面，用空格与原有的值隔开

* sudo grub-mkconfig -o /boot/grub/grub.cfg

* 重启，mount查看输出如下：

  ```
  
  cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate)
  
  ```

至此安装完毕，podman info输出如下

```
:~$ podman info
host:
  arch: amd64
  buildahVersion: 1.15.2
  cgroupVersion: v2
  conmon:
    package: 'conmon: /usr/libexec/podman/conmon'
    path: /usr/libexec/podman/conmon
    version: 'conmon version 2.0.20, commit: unknown'
  cpus: 4
  distribution:
    distribution: ubuntu
    version: "20.10"
...
ociRuntime:
    name: crun
    package: 'crun: /usr/bin/crun'
    path: /usr/bin/crun
    version: |-
      crun version 0.14.1
      commit: 598ea5e192ca12d4f6378217d3ab1415efeddefa
      spec: 1.0.0
      +SYSTEMD +SELINUX +APPARMOR +CAP +SECCOMP +EBPF +YAJL
  os: linux
  remoteSocket:
    exists: true
    path: /run/user/1000/podman/podman.sock
  rootless: true
  slirp4netns:
    executable: /usr/bin/slirp4netns
    package: 'slirp4netns: /usr/bin/slirp4netns'
    version: |-
      slirp4netns version 1.0.1
      commit: 6a7b16babc95b6a3056b33fb45b74a6f62262dd4
      libslirp: 4.3.1
...
```

## 提示未找到INIT的二进制程序

这个是在gentoo上发现的问题，一并贴在这里吧。个人的理解是这里可以指定初始化容器时要执行的程序。以前应该是调用初始化脚本，现在容器初始化已经标准化了(conmon包?仅个人猜测)。于是这就相当于一个hook，可以供用户插入自己的初始化过程。不使用它的话直接设置为false即可。
```
$> podman run --init -ti cp /sbin/init
Error: container-init binary not found on the host: stat /usr/libexec/podman/catatonit: no such file or directory
```
参考[[这里](https://unix.stackexchange.com/questions/619212/podman-run-with-init-gives-me-error-container-init-binary-not-found-on-the-h)], 该问题我也给了自己的答案：
修改container的配置，有两个地方可以修改:/etc/containers/containers.conf(对所有用户生效),或者是: ${HOME}/.config/containers/containers.conf(仅对当前用户生效)
[containers]
init = false