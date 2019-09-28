# Heat templates and encoding <!-- .element: class="hidden" -->

<!-- Note -->
My third herring for you has to do with Heat templates. 


## Encoding error <!-- .element: class="hidden" -->

<!-- Note -->
What I’m doing here is fire up a Heat template, and I get a
nondescript HTTP 500. However, `--debug` will clarify that we are
dealing with a server-side encoding error. In other words: the Heat
API endpoint, not the client, is complaining that it’s been given a
template with an invalid encoding. That sounds buggy, because if it
actually *was* an incorrectly-encoded template, the Heat client should
have caught that.

And the additional information that we are getting here is pretty
useless. We’re given an exact character that Heat is complaining
about, but that one is definitely not incorrectly encoded.

The funny part about this one is that I ran into it without making any
changes to my template, which had previously worked quite all
right. The only thing that *had* changed, when I first saw this
problem, was that the OpenStack region I was running against had just
been upgraded from Ocata to Pike. And I did have another Ocata region
available, where the template ran fine, and I had other Pike-and-later
regions, where it broke.

So, surely this is a regression that somehow slipped past all the
gates and CI checks?

Well, I can tell you that I spent some rather significant time working
this one out, but in the end this turned out to be yet another red
herring.

You may recall that in Heat templates, we can use a function called
[`str_replace`](https://docs.openstack.org/heat/latest/template_guide/hot_spec.html#str-replace)
for string templating. Here’s an example:


## str_replace example <!-- .element: class="hidden" -->
```yaml
resources:
  my_instance:
    type: OS::Nova::Server
    # general metadata and properties ...
	
outputs:
  Login_URL:
    description: The URL to log into the deployed application
    value:
      str_replace:
        template: http://host/MyApplication
        params:
          host: { get_attr: [ my_instance, first_address ] }
```

<!-- Note -->
In this example, the string `host` in the `template` parameter is
replaced with the IP address of a Nova instance, resulting in a usable
URL that can then be retrieved with `openstack stack output show
Login_URL`. Fairly straightforward.

Apart from `get_attr`, you can of course use other functions to
construct the parameter value itself:

* `get_param`, to use a stack parameter,
* `get_resource`, to use another stack resource’s UUID,

etc.

Now, what happens here in the parameter substitution is just a simple
string replacement. That means you can name your parameters anything,
you are not required to use any variable marker prefix (like `$`
would be in bash). But that quickly makes templates very
unreadable. So most people *do* use some sort of a prefix, although
there’s not really much of a convention there. So:


## str_replace template: $ <!-- .element: class="hidden" -->
```yaml
resources:
  my_instance:
    type: OS::Nova::Server
    # general metadata and properties ...
	
outputs:
  Login_URL:
    description: The URL to log into the deployed application
    value:
      str_replace:
        template: http://$host/MyApplication
        params:
          '$host': { get_attr: [ my_instance, first_address ] }
```

<!-- Note -->
Some people do this (using `$`)...


## str_replace template: <% %> <!-- .element: class="hidden" -->
```yaml
resources:
  my_instance:
    type: OS::Nova::Server
    # general metadata and properties ...
	
outputs:
  Login_URL:
    description: The URL to log into the deployed application
    value:
      str_replace:
        template: http://<%host%>/MyApplication
        params:
          '<%host%>: { get_attr: [ my_instance, first_address ] }
```

<!-- Note -->
... this is frequently found for people coming from JSP or ASP.NET
(using some variation of `<%`)...


## str_replace template: CAPS <!-- .element: class="hidden" -->
```yaml
resources:
  my_instance:
    type: OS::Nova::Server
    # general metadata and properties ...
	
outputs:
  Login_URL:
    description: The URL to log into the deployed application
    value:
      str_replace:
        template: http://HOST/MyApplication
        params:
          HOST: { get_attr: [ my_instance, first_address ] }
```

<!-- Note -->
... some people just use capitals. This is essentially up to you. The
documentation also doesn’t mandate anything — as long as it’s valid
YAML (and of course, that’s includes things like proper encoding and
so on).

So, I’m going to show you a code snippet of one of my Heat templates
that used to run, perfectly fine, in all OpenStack releases up to
Ocata:


## str_replace example: apt_mirror <!-- .element: class="hidden" -->
```yaml
parameters:
  ubuntu_mirror:
    type: string
    description: Ubuntu package archive mirror
    default: us.archive.ubuntu.com
  [...]

resources:
  base_config:
    type: "OS::Heat::CloudConfig"
    properties:
      cloud_config:
        apt_mirror:
          str_replace:
            template: http://{mirror}/ubuntu
            params:
              "{mirror}": { get_param: ubuntu_mirror }
  [...]
```

<!-- Note -->
So what this does is, it simply takes a stack parameter named
`ubuntu_mirror`, and then it injects that into an instance’s
configuration via a `CloudConfig` resource, so that depending on which
OpenStack region I launch this stack in, I can select a suitable
Ubuntu mirror.

And like I said, this particular template worked just fine up until
Ocata, but as soon as I try to fire it up on any later release, I got
that HTTP 500.


## Encoding fixed <!-- .element: class="hidden" -->

```yaml
resources:
  base_config:
    type: "OS::Heat::CloudConfig"
    properties:
      cloud_config:
        apt_mirror:
          str_replace:
            template: http://%mirror/ubuntu
            params:
              "%mirror": { get_param: ubuntu_mirror }
  [...]
```

<!-- Note -->
In reality, this is all it took. Using the `%` prefix did the
trick. No Heat API problem. And to this day I still have no idea what
exactly made this break, specifically, between Ocata and Pike — but
_something_ surely did.
