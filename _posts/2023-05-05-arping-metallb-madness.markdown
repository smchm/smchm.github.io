---
layout: post
title:  "Debugging wandering IPs with arping"
date:   2023-05-05 08:57:23 +0100
categories: [metallb, kubernetes]
---
We have a number of on premises [Kubernetes](https://kubernetes.io/) clusters on virtual hardware which utilise [MetalLB](https://metallb.org/) for load balancing. New [configuration options](https://metallb.org/configuration/_advanced_l2_configuration/) enables the coupling of specific IP address pools to named Kubernetes nodes, this functionality prompted an upgrade.

During the upgrade of a specific cluster our services started flapping. We noticed IPs in the range were constantly being reassigned to different nodes whilst also intermittently failing, here demonstrated by `sshing` into the node via the IP assigned to it by MetalLB and retrieving the hostname:

(code examples are for the [fish shell](https://fishshell.com/]))
{% highlight bash %}

for ip in (host vip-pool.hostname | grep address | awk '{print $4}' | sort) ; 
    echo -n "$ip   " && ssh -o ConnectTimeout=2 -o "UserKnownHostsFile=/dev/null" \
        -o LogLevel=ERROR root@$ahost "cat /etc/hostname"; 
end;
10.34.11.190   kubenode001
10.34.11.191   kubenode002
10.34.11.192   ssh: connect to host 10.34.11.192 port 22: Connection timed out
10.34.11.193   ssh: connect to host 10.34.11.193 port 22: Connection timed out
10.34.11.194   ssh: connect to host 10.34.11.194 port 22: Connection timed out
10.34.11.195   kubenode003
10.34.11.196   kubenode004
10.34.11.197   kubenode005
10.34.11.198   ssh: connect to host 10.34.11.198 port 22: Connection timed out
10.34.11.199   kubenode006

# and again

for ip in (host vip-pool.hostname | grep address | awk '{print $4}' | sort) ;
    echo -n "$ip   " && ssh -o ConnectTimeout=2 -o "UserKnownHostsFile=/dev/null" \
        -o LogLevel=ERROR root@$ahost "cat /etc/hostname";                                    
end;
10.34.11.190   ssh: connect to host 10.34.11.190 port 22: Connection timed out
10.34.11.191   ssh: connect to host 10.34.11.191 port 22: Connection timed out
10.34.11.192   kubenode001
10.34.11.193   kubenode002
10.34.11.194   ssh: connect to host 10.34.11.194 port 22: Connection timed out
10.34.11.195   kubenode003
10.34.11.196   ssh: connect to host 10.34.11.196 port 22: Connection timed out
10.34.11.197   kubenode004
10.34.11.198   ssh: connect to host 10.34.11.198 port 22: Connection timed out
10.34.11.199   ssh: connect to host 10.34.11.199 port 22: Connection timed out

{% endhighlight %}

It seems IP addresses are constantly being reassigned to different nodes, and this should happen, according to the [docs](https://metallb.org/concepts/layer2/#load-balancing-behavior) because

> . . . the leader node [has] failed for some reason, failover is [then] automatic: the failed node is detected using memberlist, at which point new nodes take over ownership of the IP addresses from the failed node.

Let's investigate further with [arping](https://github.com/ThomasHabets/arping)

{% highlight bash %}
for suffix in $(seq 190 199) ; do arping -I ens192 -c 1 10.34.11.$suffix; done;
ARPING 10.34.11.190 from 10.34.11.230 ens192
Unicast reply from 10.34.11.190 [00:50:56:AB:47:53]  0.695ms
Unicast reply from 10.34.11.190 [00:50:56:AB:62:40]  0.730ms
Unicast reply from 10.34.11.190 [00:50:56:AB:D4:6D]  0.740ms
Unicast reply from 10.34.11.190 [00:50:56:AB:89:59]  0.837ms
Unicast reply from 10.34.11.190 [00:50:56:AB:7A:9F]  0.872ms
Sent 1 probes (1 broadcast(s))
Received 5 response(s)
ARPING 10.34.11.191 from 10.34.11.230 ens192
Unicast reply from 10.34.11.191 [00:50:56:AB:62:40]  0.700ms
Unicast reply from 10.34.11.191 [00:50:56:AB:47:53]  0.745ms
Unicast reply from 10.34.11.191 [00:50:56:AB:D4:6D]  0.753ms
Unicast reply from 10.34.11.191 [00:50:56:AB:89:59]  0.827ms
Unicast reply from 10.34.11.191 [00:50:56:AB:86:6B]  0.852ms
Sent 1 probes (1 broadcast(s))
Received 5 response(s)
ARPING 10.34.11.192 from 10.34.11.230 ens192
Unicast reply from 10.34.11.192 [00:50:56:AB:47:53]  0.643ms
Unicast reply from 10.34.11.192 [00:50:56:AB:62:40]  0.666ms
Unicast reply from 10.34.11.192 [00:50:56:AB:D4:6D]  0.732ms
Unicast reply from 10.34.11.192 [00:50:56:AB:89:59]  0.800ms
Unicast reply from 10.34.11.192 [00:50:56:AB:38:AC]  0.981ms
Unicast reply from 10.34.11.192 [00:50:56:AB:A4:FC]  1.034ms 
...
{% endhighlight %}

Here we have responses coming from multiple interfaces for the same IP. Grepping for the above MAC addresses in the output of `ip a` on the cluster nodes revealed many were for unconfigured virtual interfaces, no wonder our services were flapping. By default MetalLB will attempt to broadcast on every interface, [up or down](https://www.reddit.com/r/kubernetes/comments/p2uwfw/comment/h8mo70a). To disable the default behaviour we need to augment our `L2Advertisement` Custom Resource configuration to specify the interfaces [we want to use](https://metallb.org/configuration/_advanced_l2_configuration/#specify-network-interfaces-that-lb-ip-can-be-announced-from), resulting in a stable service:

{% highlight bash %}

for ip in (host vip-pool.hostname | grep address | awk '{print $4}' | sort) ;
    echo -n "$ip   " && ssh -o ConnectTimeout=2 -o "UserKnownHostsFile=/dev/null" \
        -o LogLevel=ERROR root@$ahost "cat /etc/hostname";
end;
10.34.11.190   kubenode001
10.34.11.191   kubenode002
10.34.11.192   kubenode003
10.34.11.193   kubenode004
10.34.11.194   kubenode005
10.34.11.195   kubenode006
10.34.11.196   kubenode007
10.34.11.197   kubenode008
10.34.11.198   kubenode009
10.34.11.199   kubenode010

{% endhighlight %}
