---
title: You May Not Need Service Discovery
description: A consideration for distributed lifecycle tokens...
thumbnail: photo-1597733336794-12d05021d510.jpg 
date: 2019-12-23 18:46:30
updated: 2020-11-05 11:06:55
tags:
    - Programming
    - DevOps
    - C Sharp
---

<!-- markdownlint-disable no-inline-html -->
![Image](photo-1597733336794-12d05021d510.jpg)<span class="caption">Image: [JJ Ying](https://unsplash.com/@jjying)</span>
<!-- markdownlint-enable no-inline-html -->

As modern application architecture is decoupled more and more into independently running services, we're all once again experiencing the pains that some of us remember from the [SOA](https://en.wikipedia.org/wiki/Service-oriented_architecture) days in the early 2000s. A popular saying in software engineering is that the two hardest problems in computer science are:

1. Cache invalidation
2. Naming things

...and then everyone typically adds a third problem, based on whatever particular stress they are going through at that very moment:

<!-- markdownlint-disable no-bare-urls -->
{% twitter https://twitter.com/codinghorror/status/506010907021828096 %}
<!-- markdownlint-enable no-bare-urls -->

As of this year, my "number three" on the list is:

<!-- markdownlint-disable ol-prefix -->
3. Constantly having to remind myself that in-memory communication between software components is _always_ easier than over the wire
<!-- markdownlint-enable ol-prefix -->

My inner-voice is there to convince me that breaking a component off of the _monolith_ (that component being what we are all calling a _microservice_ now) probably introduces a range of new problems that can overshadow the original perceived value of decomposing the application in the first place. Even load-balanced monolithic application servers have SOA-like problems that are still there even if the monolith goes away. At [MindTouch](https://mindtouch.com), we ran into one that had been hiding in plain sight for nearly a decade.

We've used a _progressive_ [blue-green deployment](https://en.wikipedia.org/wiki/Blue-green_deployment) for many years to release new software on our shared, multi-tenant SaaS infrastructure. The approach is progressive in-so-far that tenants are switched to new software in batches (to monitor for problems and lessen their impact) as opposed to flipping a switch for all tenants at once.

![Image](image21.png)

Tenants are identified by hostname (ex: `foo.mindtouch.us`, `bar.mindtouch.us`, etc). Incoming requests are routed to the appropriate pool of EC2-hosted application servers for the requested tenant. If a blue-green deployment is in progress, and the tenant is queued to switch to the new software release but has not yet been moved over to it, the load balancer can still route correctly. This is due to the presence of a site configuration authority: a centralized database of all tenants and their deployments, plus an internal tenant manager application that writes to this database. Deployments are simply a list of EC2 IP addresses that represent the application servers in a particular pool.

```json
{
    "release_20130620": [
        "161.21.53.204",
        "178.3.205.241",
        "211.244.229.69",
        "137.62.44.231"
    ],
    "release_20130617": [
        "210.124.193.170",
        "177.113.206.133",
        "222.179.114.209",
        "7.147.241.158"
    ]
}
```

![Image](image22.png)

Blue-green can certainly mitigate risk in this "load-balanced application server" architecture. Any software update, regardless of whatever model or methodology you use, will always introduce risk. This particular approach makes it tolerable for us and our users. Due to the requirements that we had when standing up this infrastructure, we chose [HAProxy](http://www.haproxy.org) over Amazon's native [load balancer](https://aws.amazon.com/elasticloadbalancing). This meant that while we _could_ place these application servers in [auto-scaling groups](https://aws.amazon.com/autoscaling), there wouldn't be any benefit if our load balancer of choice and our home-grown deployment model couldn't take advantage of it. If our service ever came under heavy load, we would have to spin-up EC2s manually, provision them with all the requirements to run our software (and install the application software itself) using [Puppet](https://puppet.com), then add the EC2 IP address to the application server pool for the live deployment. Historically, this was required so infrequently that automating these steps didn't seem necessary.

Until we had a real SaaS business going...

This year we completed a migration of all application servers and event-driven microservices to containers hosted on [Amazon EKS](https://aws.amazon.com/eks) (k8s). The many reasons behind the switch are deserving of a case study, but the key point that's relevant to this post is that we could _quickly_ auto-scale application servers (now in k8s _pods_) _and_ it was becoming necessary to do so. As an added benefit, all load balancing between deployments could be managed within k8s, making the old deployment model of static IP address lists obsolete.

There was one big problem. The fore-mentioned internal tenant manager can _restart_ tenants - which is required for certain kinds of configuration changes. Each application server, in every container, keeps an in-memory record of the configurations for every tenant that it's had to serve. Keeping an in-memory copy allows any application server to quickly handle a request for any tenant (so specific tenants need not be pegged to specific containers). In order to restart a tenant, the tenant manager would send a restart HTTP payload to an internal API endpoint on each application server. With k8s managing and auto-scaling all containers, this application had no idea which IP addresses to target - effectively breaking this capability.

{% blockquote Everyone ever with a one to many or many to many networked service communication problem %}
We need service discovery!
{% endblockquote %}

The faulty assumption, out of the gate, is that this tenant manager still needs to know _where_ the downstream application is running. When faced with challenges, we can often gravitate towards what is known or what we are comfortable with. We presume that other options are going to be difficult to achieve. Why not? We obviously chose the simplest option when we first built this! With that presumption in place, we are likely ignoring an _even simpler_ and possibly more elegant solution - because in the last few years we've become smarter and better at our craft.

All the tenant manager needs to know about is one location: where to write a big message on the side of a wall for someone to read later. The message is for any application server that reads it to restart a specific tenant - and every application server knows to check the wall before handling an incoming request, just in case there is new information about the intended tenant for the request. We implemented this with a _distributed lifecycle token_.

A distributed lifecycle token is an arbitrary value stored in an authoritative location that downstream applications and services can use to detect upstream changes. If downstream components store the value of the token and later detect that their stored value no longer matches the authoritative source, they can infer some sort of meaning from that. In our use case, they know that the tenant manager has requested the restart of a tenant. Allowing downstream components to _eventually_ take action on, or be triggered by, a write to a data store, an event stream, or a similar authoritative location is a great example of event-driven and [reactive programming](https://en.wikipedia.org/wiki/Reactive_programming) in use.

First, we need a connection to a centralized, in-memory data store (such as [Memcached](https://memcached.org), [Redis](https://redis.io), etc.) that can be accessed by any application server. Any tokens in this data store need to contain some sort of tenant-specific context. The following example is _not_ meant to be representative of a full-featured Memcached client, but rather the minimum required to demonstrate this implementation.

```csharp
public class MemcachedClient : IMemcachedClient {

    //--- Fields ---
    private readonly IMemcachedClient _client;
    private readonly string _tenantId;

    //--- Methods ---
    public MemcachedClient(string tenantId, IMemcachedClient client) {
        _client = client;
        _siteId = siteId;
    }

    public object Get(string key) {
        return _client.Get(NormalizeKey(key));
    }

    public IDictionary<string, object> Get(IEnumerable<string> keys) {
        var result = _client.Get(keys.Select(NormalizeKey).ToArray());
        return result.ToDictionary(kv => DenormalizeKey(kv.Key), kv => kv.Value);
    }

    private string NormalizeKey(string key) {
        return String.Format("{0}:{1}", _siteId, key);
    }

    private string DenormalizeKey(string key) {
        return key.Substring(_siteId.Length + 1);
    }
}
```

Next, we have libraries that manage the initialization and fetching of tokens using our tenant-contextual Memcached client.

```csharp
public interface IDistributedTokenProvider {

    //--- Methods ---
    IDictionary<string, string> GetDistributedTokens(IEnumerable<string> keys);
    string GetDistributedToken(string key);
    string InitToken(string key);
    void DistributeToken(string key, string token, TimeSpan ttl);
    string InitDistributedToken(string key, TimeSpan ttl);
}

public class DistributedTokenProvider : IDistributedTokenProvider {

    //--- Fields ---
    private IMemcachedClient _memcache;

    //--- Constructors ---
    public DistributedTokenProvider(IMemcachedClient memcache) {
        _memcache = memcache;
    }

    //--- Methods ---
    public IDictionary<string, string> GetDistributedTokens(IEnumerable<string> keys) {
        if(keys.Count() == 0) {
            return new Dictionary<string, string>();
        }
        keys = keys.Distinct().ToArray();
        var result = _memcache.Get(keys);
        return result.ToDictionary(kv => kv.Key, kv => SysUtil.ChangeType<string>(kv.Value));
    }

    public string GetDistributedToken(string key) {
        return _memcache.Get<string>(key);
    }

    public string InitToken(string key) {

        // we have an internal utility for generating random memcache-friendly key strings,
        // point is, make the key safe and unique
        return key + ":" + StringUtil.CreateAlphaNumericKey(8);
    }

    public void DistributeToken(string key, string token, TimeSpan ttl) {
        var expires = GlobalClock.UtcNow + ttl;
        if(expires != DateTime.MaxValue) {
            _memcache.Store(StoreMode.Set, key, token, expires);
        } else {
            _memcache.Store(StoreMode.Set, key, token);
        }
    }

    public string InitDistributedToken(string key, TimeSpan ttl) {
        var token = InitToken(key);
        DistributeToken(key, token, ttl);
        return token;
    }
}
```

Next, we wrap our general-purpose distributed token logic in an easy to call utility. We can use this both in the tenant manager as well as the downstream application servers.

```csharp
public class TenantLifecycleTokenProvider {

    //--- Class Fields ---
    private const string TOKEN_KEY = "TOKEN_TENANT";
    private static readonly TimeSpan TOKEN_TTL = TimeSpan.FromSeconds(60 * 60 * 24 * 7);

    //--- Fields ---
    private IDistributedTokenProvider _tokenProvider;
    private string _token = null;

    //--- Constructors ---
    public TenantLifecycleTokenProvider(IDistributedTokenProvider tokenProvider) {
        _tokenProvider = tokenProvider;
    }

    //--- Methods ---
    public string GetToken() {
        if(_token == null) {
            _token = _tokenProvider.GetDistributedToken(TOKEN_KEY);
        }
        return _token;
    }

    public string InitToken() {
        _token = _tokenProvider.InitDistributedToken(TOKEN_KEY, TOKEN_TTL);
        return _token;
    }
}
```

Instead of sending an HTTP payload to different IP addresses in a pool of application servers, now the tenant manager simply does this:

```csharp
var tenantId = '12345';
var lifecycleProvider = new TenantLifecycleTokenProvider(_tokenFactory.NewDistributedTokenProvider(tenantId));
lifecycleProvider.InitToken();
```

A downstream application server, when it handles an incoming request for a tenant, first checks if it can lookup an in-memory copy of the tenant's configuration. If not, it fetches it from the site configuration authority _and_ looks up the tenant's lifecycle token. If the lifecycle token for this tenant is not yet initialized (it may not have been if the tenant manager didn't need to restart the tenant), then the application server initialize it, so that other application servers in the pool can know the current lifecycle state of the tenant.

```csharp
var tenantId = '12345';
var tenantConfiguration = siteConfigurationAuthorityConnector.getTenantConfiguration(tenantId);
var lifecycleToken = lifecycleTokenProvider.GetToken();
if(lifecycleToken == null) {
    lifecycleToken = lifecycleTokenProvider.InitToken();
}
var tenant = new Tenant(tenantConfiguration, lifecycleToken);
tenants.Add(tenant.Id, tenant);
```

When handling an incoming request, the application server checks if the tenant needs to be restarted before fulfilling the request. If the distributed lifecycle token has changed since we last read it and no longer matches the tenant's lifecycle token, we know that the tenant manager is asking for a tenant restart.

```csharp
var tenantId = '12345';
var tenant = tenant.Get(tenantId);
var lifecycleToken = lifecycleTokenProvider.GetToken();
if(tenant.LifecycleToken != lifecycleToken) {
    tenant = null;
}
```

In our particular architecture, nullifying the tenant at this stage triggers the application server to instantiate a new `Tenant` object and register it in the lookup, as seen above. This process is repeated for all application servers. The addition or reduction of containers or pods does not affect the tenant manager's ability to restart tenants.

I am absolutely _not_ a critic of service discovery in principal. When necessary to facilitate _direct communication_ with components on a network, it can be a life-saver. However, if your use case does not require direct communication and eventual consistency among downstream data, and application state is acceptable for your use case, consider reactive programming solutions. Everything will be ok (eventually)!

...and if it's not, you can always deploy systems to handle those situations out-of-band:

{% youtube mxKhbU_ToMs %}
