---
layout: post
title: "Using Ceph RBD snapshots with OpenStack Liberty"
modified: 2017-05-01 11:00:00 -0800
tags: [ openstack, ceph, rbd, liberty]
image:
  feature:
  credit:
  creditlink:
comments:
share: "Using Ceph RBD snapshots with OpenStack Liberty"
---

I recently had an interesting issue with the OpenStack platform I manage. This platform hosts several public services at Nuage Networks, [Nuage Networks eXperience](http://nuagex.io) is one of them. NuageX environments are composed of several virtual machines, all deployed from snapshots. But snapshots of large RBD disks (>100GB) were taking hours to complete.

I have started to look into improving these snapshots and Sebastien's blog post ([OpenStack Nova snapshots on Ceph RBD](https://www.sebastien-han.fr/blog/2015/10/05/openstack-nova-snapshots-on-ceph-rbd/)) gave me THE solution.
One problem, I am running OpenStack Liberty and this feature was officially merged on Mitaka.

Upgrading to Mitaka was not an option so I have decided to back port this feature on Liberty, I hope it will help some of you who are still running an old OpenStack version. This patch is available as a Github Gist: [Nova Liberty Ceph RBD Snapshots patch](https://git.io/v98IY)). I have tested it, you can use it, but it's at your own risk.

Before applying the patch on your compute nodes, check that:
  - show_image_direct_url=True in /etc/glance/glance-api.conf on your controller
  - show_multiple_locations=True in /etc/glance/glance-api.conf on your controller
  - ceph client on compute nodes must have write permission on glance-images pool. You can update the permissions with: {% highlight shell %}ceph auth caps client.compute mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=vms, allow rwx pool=glance-images'{% endhighlight %}

To apply the patch locally on your nova compute nodes:

{% highlight shell %}
curl -L https://git.io/v98IY -o '/tmp/nova-liberty-rbd-direct-snapshot.patch'
cd /usr/lib/python2.7/site-packages/
patch -p1  < /tmp/nova-liberty-rbd-direct-snapshot.patch
{% endhighlight %}

  > Note: If you are using Ubuntu, replace /usr/lib/python2.7/site-packages/ by /usr/lib/python2.7/dist-packages/.

Finally, restart the OpenStack Nova Compute service with: {% highlight shell %}systemctl restart openstack-nova-compute{% endhighlight %}

This patch helped me decrease snapshot duration from several hours to a few minutes (~5 minutes) for large RBD disks.

I really want to thank Sebastien for his great and detailed blog post and recommend reading Sebastien's blog as it's a great source of information for Ceph users/administrators!
