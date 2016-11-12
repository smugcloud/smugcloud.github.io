---
layout: post
title:  "Building a Container With Syscalls"
date:   2016-11-07
categories: jekyll update
---
Recently, I have been playing with [runc](https://github.com/opencontainers/runc), [lxc](https://linuxcontainers.org/), and of course Docker, when I became curious about how those containers were actually created.  This post will follow my journey through the various Linux components to create a container, including network connectivity to the outside world.  All commands will be run in a Ubuntu 16.04 VM [vagrant box](https://vagrantcloud.com/ubuntu/boxes/xenial64/versions/20161109.1.0).

To start, I watched the wonderful [session](https://www.youtube.com/watch?v=sK5i-N34im8) from the tinkerer-extraordinaire, Jérôme Petazzoni.  This post is heavily influenced by his fantastic work.

At it's core, Docker is utilizing a variety of Kernel technologies to isolate processes, including the network stack.  To start, let's create a private filesystem to ensure persistent mount namespacing.

```
mount --make-private /
```

Next, create the directory structure that will hold the container filesystem.

{% highlight bash %}

mkdir -p images containers
mkdir -p images/alpine

{% endhighlight %}

Let's follow Jérôme's lead and base all of this off of an Alpine image.

{% highlight bash %}
CID=$(docker run -d alpine true)
docker export $CID | tar -C images/alpine/ -xf-
cp -al images/alpine containers/alpine
{% endhighlight %}

So now we have the file structure to begin isolating this container.  Let's start the magic of [unshare](https://linux.die.net/man/2/unshare)

{% highlight bash %}
unshare --mount --net --pid --uts --ipc --fork bash
hostname mycontainer
exec bash
mount -t proc none /proc
{% endhighlight %}


Now if you issue `ps`, you will see only a subset of processes actually running in this namespace.  You have a container!  Well, mostly.

{% highlight bash %}

root@mycontainer:/home/ubuntu# ps
  PID TTY          TIME CMD
    1 pts/0    00:00:00 bash
   22 pts/0    00:00:00 ps
   {% endhighlight %}


Now some quirks of the kernel come into play.  We need to bind mount the directory we want to work with, in order to utilize the `pivot_root` syscall.  Thanks to the folks contributing to runc/LXC, there is a [trick](https://github.com/opencontainers/runc/blob/master/libcontainer/rootfs_linux.go#L598) with `pivot_root` below.

{% highlight bash %}

mount --bind /home/ubuntu/containers/alpine /home/ubuntu/containers/alpine
mkdir /alpine
mount --move /home/ubuntu/containers/alpine /alpine
cd /alpine/
pivot_root . .
## And you can see the filesystem looks different now
root@mycontainer:/alpine# cd /
root@mycontainer:/# ls
bin      dev      etc      home     lib      linuxrc  media    mnt      proc     root     run      sbin     srv      sys      tmp      usr      var
{% endhighlight %}


Ok great, now we have a file structure of a working Alpine distribution in our tree.  Since this is a completely isolated set of namespaces, there is no networking stack from within our container.

{% highlight bash %}

root@mycontainer:/# ifconfig -a
ifconfig: /proc/net/dev: No such file or directory
{% endhighlight %}

So let's set up network.  Open up another terminal session to your machine and use the following commands.

{% highlight bash %}

#Get the PID of your unshare process and store it for later
CPID=$(pidof unshare)
ip link add name veth1 type veth peer name c-veth
ip link set c-veth netns $CPID
ip link set veth1 master docker0 up
{% endhighlight %}


With these commands, we have just set up a pair of virtual interfaces, connected one end to our `docker0` bridge, and the other end to our network namespace.

We still need to set up the rest of the network stack from within the container.  If you inspect `docker0` in the global namespace, you'll see the details of that bridge

{% highlight bash %}

root@ubuntu-xenial:/# ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:87:91:84:81
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          {% endhighlight %}

Mask 255.255.0.0 translates to 65536 IP's in the 172.17.0.0 - 172.17.255.255 range.  For the purpose of this article, we'll just pick a static IP.  Docker handles this much more elegantly and will assign containers a free address in the range.  From within your container terminal, execute the commands below.

{% highlight bash %}

ip link set lo up
ip link set c-veth up
ip addr add 172.17.0.3/24 dev c-veth
ip route add default via 172.17.0.1
{% endhighlight %}


If we look at the container networking stack again, we can see that we have a working network stack, and can even ping a nameserver.

{% highlight bash %}

root@mycontainer:/# ifconfig -a
c-veth    Link encap:Ethernet  HWaddr B2:95:88:A3:BA:48
          inet addr:172.17.0.3  Bcast:0.0.0.0  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1

root@mycontainer:/# ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=61 time=16.415 ms
64 bytes from 8.8.8.8: seq=1 ttl=61 time=14.559 ms
64 bytes from 8.8.8.8: seq=2 ttl=61 time=54.255 ms
{% endhighlight %}


While we can directly access IP's, it would be nice to use DNS entries to fully round off this stack.  So let's add a `resolv.conf`

{% highlight bash %}

echo "nameserver 8.8.8.8" > /etc/resolv.conf
root@mycontainer:/# ping google.com
PING google.com (172.217.3.206): 56 data bytes
64 bytes from 172.217.3.206: seq=0 ttl=61 time=20.816 ms
64 bytes from 172.217.3.206: seq=1 ttl=61 time=16.588 ms
64 bytes from 172.217.3.206: seq=2 ttl=61 time=23.470 ms
{% endhighlight %}


Excellent.  Does this mean we can update the Alpine repositories?

{% highlight bash %}

root@mycontainer:/# apk update
fetch http://dl-cdn.alpinelinux.org/alpine/v3.4/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.4/community/x86_64/APKINDEX.tar.gz
v3.4.6-2-g20654dd [http://dl-cdn.alpinelinux.org/alpine/v3.4/main]
v3.4.4-21-g75fc217 [http://dl-cdn.alpinelinux.org/alpine/v3.4/community]
OK: 5975 distinct packages available
{% endhighlight %}


Awesome! we just created a Linux container using various CLI utilities that interact with system calls.
