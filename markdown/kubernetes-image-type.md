<!-- .slide: data-background-image="images/magnum.svg" data-background-size="contain" -->
# Magnum, Kubernetes, and image properties <!-- .element: class="hidden" -->

<!-- Note -->
Here’s another one that’s interesting. In Magnum, your prerequisites
to run a Kubernetes cluster are

1. You need to have a Glance _image_ for one of the operating system
   platforms that Kubernetes supports. There are several, but Fedora
   Atomic is really the best-tested and most widely used.

2. You need a Magnum cluster _template_ that references that image.

3. Finally, you can use the template to spin up the cluster.


<!-- .slide: data-background-color="#121314" -->
## Kubernetes 1.15.3 on Fedora Atomic 27 <!-- .element: class="hidden" -->

<iframe src="https://asciinema.org/a/ijCMCBowtxaZvpZVmHwHOqwZv/embed?size=big&rows=19&cols=60&theme=tango" class="stretch"></iframe>

<!-- Note --> 
OK, so let’s look at that. Here’s my image. It’s a Fedora Atomic Host
27 image, which should be totally supported to deploy Kubernetes
1.15.3 with Magnum.

And here is my cluster template. It sets the cluster orchestration
engine (COE) to `kubernetes`, and the `kube_tag` label to v1.15.3,
which means that Magnum installs that Kubernetes release.

All right, wonderful. My cluster is spinning up exactly as expected.

*But*, I really want to use a more current Fedora Atomic image
instead, such as Fedora Atomic Host 29. 


<!-- .slide: data-background-color="#121314" -->
## Fedora Atomic 29 image <!-- .element: class="hidden" -->

<iframe src="https://asciinema.org/a/IDF78b4U70LQIQnYwuFLBZ0cT/embed?size=big&rows=19&cols=60&theme=tango" class="stretch"></iframe>

<!-- Note --> 
Now to do that, I‘ve uploaded a Fedora Atomic 29 image. Again,
deploying Kubernetes off of this should totally work.

But this is weird. Everything is set up exactly as it should be, and
what I’m getting is this nondescript HTTP 400 error.


<!-- .slide: data-background-color="#121314" data-background-image="//http.cat/400.jpg" data-background-size="contain" -->
## HTTP 400 from Magnum <!-- .element: class="hidden" -->

<!-- Note --> 
So HTTP 400 is Bad Request. Clearly, that means that something is
wrong with my API call, right? And since I’ve been making all my API
calls exactly by the book, the first problem to assume would be a bug
in the Magnum client library, or the `openstack` client, right?

Well, wrong again. Another red herring.


<!-- .slide: data-background-color="#121314" -->
## Fedora 29 image properties <!-- .element: class="hidden" -->

<iframe src="https://asciinema.org/a/LdEJRK58H4eWqSFP9uSW5VyCa/embed?size=big&rows=19&cols=60&theme=tango" class="stretch"></iframe>

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

Once we set this variable, we’re ready to create our cluster template.

And once we’ve got the template, we can fire up a new cluster:

... and then we can use Kubernetes from there.

