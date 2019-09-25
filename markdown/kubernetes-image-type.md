# Magnum, Kubernetes, and image properties <!-- .element: class="hidden" -->

<!-- Note -->
Here’s another one that’s interesting. In Magnum, your prerequisites
to run a Kubernetes cluster are

1. You need to have a Glance _image_ for one of the operating system
   platforms that Kubernetes supports. There are several, but Fedora
   Atomic is really the best-tested and most widely used.

2. You need a Magnum cluster _template_ that references that image.

3. Finally, you can use the template to spin up the cluster.


## Fedora Atomic 29 image <!-- .element: class="hidden" -->

<!-- Note --> 
OK, so let’s look at that. Here’s my image. It’s a Fedora Atomic Host
29 image, which should be totally supported to deploy Kubernetes
1.15.3 with Magnum.


## Kubernetes 1.15.3 template <!-- .element: class="hidden" -->

<!-- Note --> 
And here is my cluster template. It sets the cluster orchestration
engine (COE) to `kubernetes`, and the `kube_tag` label to v1.15.3,
which means that Magnum installs that Kubernetes release.

So far, so good, right?


## Failed Kubernetes 1.15.3 spin-up <!-- .element: class="hidden" -->

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
