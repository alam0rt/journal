# Pushing Prometheus, or how to blunt your instruments

The Prometheus authors make it clear: do not use the Pushgateway as a
way to circumvent NAT or similar network topologies. The reason?
Well the Prometheus authors don't really say why. While I was building
a monitoring platform for 850+ of our Linux boxes sitting inside the
customer's network, we assumed we'd be provisioning a Prometheus server
inside too, and then federating it to our core Prometheus service sitting in
EKS. Alas, the customer's requirements changed, "we wan't these boxes to operate
like an IoT device, it should be plug and play!"
Fair enough, I was annoyed that the requirements had changed, but this was
an oppourtunity to learn. I was already up to my neck in Prometheus and
the Chief Architect wanted to stick with it, cool.

We broke out the whiteboard markers and quickly (and naively) decided that
we should proof-of-concept a method where we run a Bash script on each machine
which did the following:

`
while true; do
  for target in "${SCRAPE_TARGETS[@]}"; do
    for pushgateway in "${PUSHGATEWAY_SERVERS[@]}"; do
      sh -c \
      "curl -s ${target}/metrics | curl --data-binary @- ${pushgateway_api}/$NODE_NAME" &>/dev/null &
    done
  done
  sleep "${PUSH_INTERVAL}"
done
`
This let us `source` a configuration file which defined our
`$PUSHGATEWAY_SERVERS` and `$SCRAPE_TARGETS`, and turned pull into push. Great.

So the first thing I did was go to [WHEN TO USE THE
PUSHGATEWAY](https://prometheus.io/docs/practices/pushing/)', and if it's not
clear from the page (hint: it is), using the Pushgateway in this manner is *not*
recommended. Namely:

  * The Pushgateway is a single point of failure.
  * You lose the `up`metric
  * The Pushgateway holds onto any metric and will not forget it.
  
The first point was okay, I was confident enough in Kubernetes to look after
the Pushgateway well enough. We didn't have any SLAs so to speak for this
project yet.

The second point though really made me *get* pushed based metrics vs pull based.
How do you know if an endpoint is down? Protip: You don't, you just made a good
guess that if you haven't heard from it in a `${PUSH_INTERVAL}` or two, that it
likely
is having problems. It is probably obvious to anyone who does monitoring, and
I honestly had read all the right blogs and books, but it never clicked properly
that monitoring boils down to having confidence in your instruments.
I'll get back to this in an example very shortly.

The third and final point made more sense to me, I understood what the issue
was with caching metrics: suddenly, metrics can go stale.

The reasons we opted for Prometheus was that
  * It is simple (simple enough at least to write our own exporters)
  * It's written in Go and the application compiles to a single binary which
    makes it nice to deploy. Plus we have Go developers on hand.
  * It offers high resolution (per millisecond) of metrics
  * Strong community, extremely important.

Well, these issues don't seem like they are too bad, or so I thought. Let me
show you why the Pushgateway being used as NAT circumvention is an anti-pattern.

What's an anti-pattern? An anti-pattern is a bit tricky to understand if you
don't really know what you're looking for (at least it took
me until recently to *get* what it meant), but I will try my best to explain it.

An anti-pattern is a solution. A bad solution, but a solution nontheless.
What makes a solution bad? In my case, the solution of using Pushgateway to
circumvent NAT + firewalling worked (as solutions should), but at the cost of
insidious and hidden (at least it was to me) *technical debt* and blunting of my instruments.

The reasons we opted for Prometheus, as I stated above were essentially:
simplicity, accuracy and community, and this is how we lost those things.

# Accuracy


Enter **CUSTOMER MACHINE**. **CUSTOMER MACHINE** is running `node-exporter`.

1) At 1:00:0000pm a Bash script suspiciously similar to the one above wakes from its
`  sleep()` and GETs the `/metrics` endpoint on localhost, this takes `0.05`
   seconds total (`$ time curl localhost:9100/metrics`)

2) At 1:00:0050pm we push to the Pushgateway, this takes about `0.35` seconds.

So stright away, we've lost our millisecond resolution. Now it's closer to half
a second accuracy with *no* gurantee, we know what time the metrics reached the
Pushgateway but not when they were created or how long they were in flight for.

3) Enter **PROMETHEUS**, this turbo-chad of a program goes down its list of targets
   and finds the Pushgateway, in the same way the Bash script gets the `/metrics`
   endpoint
   of `node-exporter`, **PROMETHEUS** GETs the `/metrics` endpoint and injests it
   into its time series database. **PROMETHEUS**, being an absolute king decides
   it will only pull once every 15 seconds.
   
Here we've lost our half-second resolution as we are only pulling every 15
seconds. We could turn this up to scrape every second, but the script which is
on each **CUSTOMER MACHINE** only pushes every 15 seconds also. If this were to
be changed, We'd have to deploy a new version of the monitoring client package,
as configuration management can't be used (customer requirement, I tried to make
a case for it but was rebuffed).

So our time series will look like:

            _____________________________

PROMETHEUS  1             2             3
TSDB        _____________________________
            |             |             ^
            |             |             |
            |_ a pushed metric at time (1:00:00pm)
               each character is a second
                          |             |
                          |_ (1:00:15pm)|
                                        |_ Now, 1:00:30pm, 3rd set of metrics in Prometheus

Now remember we said that the Pushgateway is lazy, but it isn't forgetful, it
will hold onto whatever metric you send it until either it is updated, or deleted.

Let's say **CUSTOMER MACHINE** stopped working at 1:00:20pm, because the Pushgateway doesn't
give a fuck, it will just hold onto whatever **CUSTOMER MACHINE** sent it. It
has no way of telling ***PROMETHEUS*** explicitly it is down (there is a metric which)
has the time that the Pushgateway last heard from **CUSTOMER MACHINE**)

I will add a bit more onto the diagram to express the down **CUSTOMER MACHINE***

              _____________________________

CUSTOMER     o---------------o-----x
MACHINE                            |
                                   |
PUSHGATEWAY  o---------------o-----|------> Pushgateway never updates
             |_______________|_____|_______
PROMETHEUS   |1             2|     |      3
TSDB         |_______________|_____|_______
             |               |     |
             |               |     |
             |_ 1st metric pushed to PGW was pushed just before the Prometheus
                server's 1:00:00pm (which might be 12:59:59:5000am actual)
                             |     |
                             |_2nd Pushed metric
                                   |
                                   |_Server crash

So we can see that what **PROMETHEUS** is reporting isn't necessarily what
is happening, and *when* these things are being reported isn't actually
when they are being reported either. It's a good estimate, but our tools are
less accurate.

So now not only do we have only 15 seconds worth of resolution. We don't even
know if the pulled metric is *fresh*, we need to rely on other metrics to
determine the age of the pushed metric. More work, more complexity and most
definetely a result of an anti-pattern.

This is why the Prometheus authors do not suggest using the Pushgateway in this
manner. While it is *alright* for general use cases, Prometheus is designed
to be *fast* and accurate. The above design is anything but, it is (relatively)
slow and not very accurate. That doesn't mean our monitor is bad, not at all.
As I mentioned earlier, we need to have confidence in our tools. How can we
be confident in our tools when it can't even tell us if **CUSTOMER MACHINE**
is down or not?

To solve this, I opted to use the following Prometheus query:

`time() - push_time_seconds() < 120`

So, if we take the current time, work out the difference between that and when
a metric was last pushed, we can get in seconds when we last heard from
**CUSTOMER MACHINE**, then we can just use common sense: "if we haven't heard
from the **CUSTOMER MACHINE** in over two minutes, consider the store unreachable"

This is good because whenever metrics are pushed to the Pushgateway, the
Pushgateway itself will automatically update the `push_time_seconds` metric,
i.e. we can assume that `time()` and `push_time_seconds` are working off
a clock which is likely the same (I've been meaning to confirm this).
Therefore I am confident in the result of the query.

Now we have a psuedo `up` metric. This is great and all but it doesn't
stop the fact that our monitoring instrumentation has been blunted.
We can never remove the inaccuracy that the Pushgateway adds.

Once again, this is *fine* if you understand the limitations of the
instruments you are using.

But coming back to the reasons we chose Prometheus, this is no longer accurate,
it's good enough, but it's not great. It's orders of magnitude slower than what
is possible, and if we wanted to tweak it to scrape more frequently, it's a
chore due to the customer's requirement of not using something like Ansible.

It's much more complex, we have a script which curls the locally running
exporters, PUTs it into the Pushgateway (which is far away from where the
**CUSTOMER MACHINE** is, exporters are best when placed close to Prometheus). We
have to use ugly PromQL to determine if a store is likely online. All of these
issues cascade down from the anti-pattern.

And finally, the community. How? Well since we are now running an exotic
configuration, we are just that, exotic. Trying to find support for weird issues
that might arise will be met with other people making the same mistakes. The
developers of Prometheus are active and responsive to the community, if I were
to go and ask them how to solve this issue with CPU rate spiking when the store 

So what is the Pushgateway meant to be used for? Short lived jobs! Short lived
being services that are ephemeral and do not lend themselves to scraping.

Notice how I used service? There is a huge difference between a service and an
instance. Many instances can create a service
