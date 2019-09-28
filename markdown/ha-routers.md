<!-- .slide: data-background-image="images/neutron.svg" data-background-size="contain" -->
# 255 HA routers per tenant <!-- .element: class="hidden" --> 

<!-- Note -->
OK. Let’s start out with something relatively straightforward: virtual
routers in Neutron.


## Router creation in a loop <!-- .element: class="hidden" --> 

<!-- Note -->
What I’m doing here is simply to create virtual routers, in a
loop. I’m operating against a single tenant, and I just add one
virtual router after another.

And that seems to all work just dandy, _until_ suddenly it doesn’t.

So let’s see what’s at fault here.


## Is this a quota issue? <!-- .element: class="hidden" --> 

<!-- Note -->
Let’s start with the obvious assumption: I’m running into an
administrator-imposed limit. Providers can set these limits through
the OpenStack **quota** system, so let’s check whether perhaps I’m
running into a quota limit. Luckily, I can always check what my quota
is, so if I look for my router quota, I see that...

... I can create 500 of them. Well, let’s see. Do I have more than 500
routers already? Nope, it’s only 255. Besides, if we actually exceed a
quota, what we ought to get back from Neutron is HTTP 413 rather than
the HTTP 500 that we’re seeing.


## Is this a configuration setting? <!-- .element: class="hidden" --> 

<!-- Note -->
So, dig further. Maybe Neutron has a configuration limit on the
maximum number of routers per tenant, just like Heat has for stacks?

Well, yes, `quota_router`, but what that sets is just the _default_
quota for routers, and it gets overridden by a quota explicitly set on
a tenant.

So no, that doesn’t get us anywhere.


## What about HA routers? <!-- .element: class="hidden" --> 

<!-- Note -->
So let’s try one thing, by way of experimentation. Let’s try and
create a router that’s got HA disabled.

Oh. Funny. Without HA it works, with HA it doesn’t. OK, what about HA
routers, how do those work?


## How do HA routers work, again? <!-- .element: class="hidden" --> 

<!-- Note -->
Way back in the OpenStack Juno release, we got high-availability
support for Neutron routers. This means that, assuming you have more
than one network gateway node that can host them, your virtual routers
will work in an automated active/backup configuration.

In effect, what Neutron does for you is that for every subnet that is
plugged into the router — and for which it therefore acts as the
default gateway — the gateway address binds to a keepalived-backed
VRRP interface. On one of the network nodes that interface is active,
and on the others it’s in standby. If your network node goes down,
keepalived makes sure that the subnets’ default gateway IPs come up on
the other node. The keepalived configuration is completely abstracted
away from the user; the Neutron L3 agent happily takes care of all of
it.

In order to enable HA routers, Neutron creates one administrative
network per tenant, over which it runs VRRP traffic. In order to tell
apart all the keepalived instances that it manages on that network, it
assigns each an individual Virtual Router ID or VRID.


## RFC 5798 and VRIDs <!-- .element: class="hidden" --> 

<!-- Note -->
And here’s the problem: RFC 5798 defines the VRID to be an 8-bit
integer. That means that if you use HA routers, then setting a router
quota over 255 is useless — Neutron will run out of VRIDs in the
administrative network, before your tenant can ever hit the quota.

And this is a hard limit; there’s really not much that Neutron can do
about this — apart from starting to spin up additional administrative
networks once it runs out of VRIDs in the first one, but that likely
would be a pretty involved change. Thus, at least for the time being,
if you want more than 255 highly-available virtual routers, you’ll
have to spread them across multiple tenants.


## What if I really don’t need HA routers?

<!-- Note -->
Well, firstly you probably do want them, really. But that aside, let’s
assume for a moment that you actually don’t. Or rather, that it’s more
important for you to have more than 255 routers in a single tenant,
than for any of them to be highly available. So you create routers
with the ha flag set to False, simple, right?

It turns out that you probably won’t be able to do that. And that’s
not because you can’t change a router’s ha flag without first
temporarily disabling it — that’s not going to hurt you much if you’ve
already decided you don’t need HA; in such a case a brief router blip
will be acceptable. Instead, it’s because (at the time of writing) the
default Neutron policy restricts setting the ha flag on a router to
admins only.

So if you want to be able to disable a router’s HA capability, you’ll
first need to override the following default entries in Neutron’s
policy.json:


## HA routers and `policy.json`


```json
{
    "create_router:ha": "rule:admin_only",
    "get_router:ha": "rule:admin_only",
    "update_router:ha": "rule:admin_only",
}
```

<!-- Note -->
... and instead set them as follows:


```json
{
    "create_router:ha": "rule:admin_or_owner",
    "get_router:ha": "rule:admin_or_owner",
    "update_router:ha": "rule:admin_or_owner",
}
```

<!-- Note -->
If your cloud service provider deploys Neutron with OpenStack-Ansible,
they can define this in the following variable:


```yaml
neutron_policy_overrides:
    "create_router:ha": "rule:admin_or_owner"
    "get_router:ha": "rule:admin_or_owner"
    "update_router:ha": "rule:admin_or_owner"
```

<!-- Note -->
Once the policy has been overridden in this manner, you should be able
to create a new router with:


```bash
openstack router create --no-ha <name>
```

<!-- Note -->
And modify an existing router’s high-availability flag with:


```bash
openstack router set --disable <name>
openstack router set --no-ha <name>
openstack router set --enable <name>
```

<!-- Note -->
There’s something else you ought to be aware of, and that’s really a
meta-red herring.

You may want to find out whether one of your routers is configured to
be highly available in the first place. You’d expect to easily be able
to do this with an openstack router show command:

Alas, what you see in the example above is indeed a highly-available
router, so why does it clearly report its ha flag as being False?

Well, that’s another consequence of that default Neutron policy, in
combination with rather unintuitive behavior by the openstack command
line client. You see, this part of the aforementioned policy


```json
{
    "get_router:ha": "rule:admin_only",
}
```

<!-- Note -->
... means you’re not even allowed to query the `ha` flag if you’re not
an admin, and when the `openstack` client is asked to display a boolean
value that the user is not allowed to even read, then it always
displays `False`.

For further details, see
<https://xahteiwi.eu/resources/hints-and-kinks/1000-routers-per-tenant-think-again/>.

