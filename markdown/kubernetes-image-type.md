# Magnum, Kubernetes, and image properties <!-- .element: class="hidden" -->

<!-- Note -->
Here’s another one that’s interesting. In Magnum, your prerequisites
to run a Kubernetes cluster are

1. You need to have a Glance _image_ for one of the operating system
   platforms that Kubernetes supports. There are several, but Fedora
   Atomic is really the best-tested and most widely used.

2. You need a Magnum cluster _template_ that references that image.

3. Finally, you can use the template to spin up the cluster.


## Fedora Atomic 27 image <!-- .element: class="hidden" -->

<!-- Note --> 
OK, so let’s look at that. Here’s my image. It’s a Fedora Atomic Host
27 image, which should be totally supported to deploy Kubernetes
1.15.3 with Magnum.


## Kubernetes 1.15.3 template <!-- .element: class="hidden" -->

<!-- Note --> 
And here is my cluster template. It sets the cluster orchestration
engine (COE) to `kubernetes`, and the `kube_tag` label to v1.15.3,
which means that Magnum installs that Kubernetes release.


## Kubernetes 1.15.3 spin-up <!-- .element: class="hidden" -->

<!-- Note --> 
All right, wonderful. My cluster is spinning up exactly as expected.

*But*, I really want to use a more current Fedora Atomic image
instead, such as Fedora Atomic Host 29. 


## Fedora Atomic 29 image <!-- .element: class="hidden" -->

<!-- Note --> 
Now to do that, I‘ve uploaded a Fedora Atomic 29 image. Again,
deploying Kubernetes off of this should totally work.


## Kubernetes 1.15.3 / F29 template <!-- .element: class="hidden" -->

<!-- Note --> 
But this is weird. Everything is set up exactly as it should be, and
what I’m getting is this nondescript HTTP 400 error.


<!-- .slide: data-background-image="//http.cat/400.jpg" data-background-size="contain" -->

<!-- Note --> 
So HTTP 400 is Bad Request. Clearly, that means that something is
wrong with my API call, right? And since I’ve been making all my API
calls exactly by the book, the first problem to assume would be a bug
in the Magnum client library, or the `openstack` client, right?

Well, wrong again. Another red herring.


## Fedora 29 image properties <!-- .element: class="hidden" -->

<!-- Note --> 
Turns out that the culprit is actually a missing property on the
image, `os_distro`. It must be set to `fedora-atomic`, otherwise the
Magnum `k8s_fedora_atomic_v1`
[driver](https://opendev.org/openstack/magnum/src/branch/master/magnum/drivers)
will refuse to use it. This is [actually
documented](https://docs.openstack.org/magnum/rocky/user/#clustertemplate),
but many Magnum users never need to use a private image, and when they
do, the missing property (and unhelpful error message) often trips
them up. 


## Fedora 29 image with `os_distro` <!-- .element: class="hidden" -->

<!-- Note --> 
Once we set this variable, we’re ready to create our cluster template:


## Fedora 29-based cluster template <!-- .element: class="hidden" -->

<!-- Note --> 
And once we’ve got the template, we can fire up a new cluster:


## Fedora 29-based cluster <!-- .element: class="hidden" -->

<!-- Note --> 
... and then we can use Kubernetes from there.

