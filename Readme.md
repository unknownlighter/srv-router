SRV Router
==========

This is a small project to provide a way to use nginx and Lua (through [OpenResty](http://openresty.org/)) to balance traffic between instances providing HTTP services in a cluster with a service discovery DNS API. 

Solutions like Consul or SkyDNS offer these APIs and provide information about services through SRV records that neither HAProxy nor Nginx can handle at the moment. This little project includes a lua script that queries the discovery DNS server and routes requests upstream HTTP servers using SRV record's data. This allows for dynamic update of the upstream server's in Nginx and relies on the service discovery layer to handle node failure detection and service registration.

SRV router was originally created for a Consul cluster, but it can be easily used for any other kind of cluster with a DNS discovery API. Alternatively you can take a look at [Consul-haproxy](https://github.com/hashicorp/consul-haproxy) and [Flynn's router](https://github.com/flynn/flynn/tree/master/router) (the latter may need some work to be used outside Flynn's PaaS).

If you have questions, suggestions or ideas about how to improve this, please open an issue to start the conversation!


Usage
-----

SRV Router is distributed as a docker container available in the docker index as a trusted build (if you want a standalone installation, the Dockerfile should give you enough information on how to do it on your own). You can run the container on boot and have it shared the host's network and listen of port 80. You can provide configuration setting the follwing environment variables from the docker run command:

* NS_IP: The DNS server IP (default: 127.0.0.1)
* NS_PORT: The port where the DNS server's listening (default: 53)
* TARGET: The DNS domain to use in service discovery queries (default: service.consul). More info below.
* DOMAINS: Comma separated list of external domains that the balanced services handle (default: lvh.me,127.0.0.1.xip.io,9zlhb.xip.io). More info below.
* KEEP_TAGS: Set to ``true`` to enable compatibility with consul tags.  Controls whether subdomains are preserved in the URL passed to the backend DNS lookup. (default: false)

For an example Systemd service definition check the misc folder in the repo.

Asumptions
----------

The lua script makes some simple assumptions about the services being routed. Basically there's a matching between a public subdomain and an internal subdomain used for service discovery. If you set your DOMAINS env variable to something like `vlipco.co,vlipco.com` and TARGET to `service.consul` a request send to the load balancer for the domain `www.vlipco.co` would be routed to the instances found by querying the DNS server for `www.service.consul`

The default value of the target variables matches the default in a Consul cluster. The domain variables is set to a series of domains that resolve to localhost, this comes handy during development (for instance if you query www.lvh.me you would be routed to www.consul.service) but should be changed to a public [sub]domain for real usage.

Missing parts
-------------

The most important missing part is implementing a simple caching table shared by all nginx workers that would prevent queries to the DNS server on each request. This should be a simple in memory cache with a very low TTL (5-10s probably) to avoid storing information about death services. By adding this the DNS server would get a constant amout of queries per minute (e.g. 12 per load balancer if a 5s TTL is used) and would reduce the latency introduced in the routing layer.

The caching feature could be very easy to implement for someone it a little more experience with Lua and Nginx, so pull requests are welcomed. Otherwise I'll try to get it done by the end of August.

There's no formal testing since I couldn't find a simple way to simulate all the behaviour. I have howerver tried this to load balance traffic in a real publicly exposed cluster without much traffic and it's seems to work fine.

Since the scope is so simply and Consul or similar tools to the heavy lifting, there shouldn't much more functionality to add, but let me know if there's something you're missing.
