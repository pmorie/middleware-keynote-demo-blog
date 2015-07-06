# Middleware Keynote Demo at Summit 2015

In this blog post, we'll deconstruct the OpenShift portions of the Middleware Keynote Demo that was
show at Red Hat Summit 2015.  If you haven't seen the keynote and demo, you can watch them
[here](https://www.redhat.com/en/about/videos/craig-muzilla-middleware-keynote-2015).  We got a lot
of positive feedback about this demo and so we thought it might be interesting if we pulled the
covers back a little and examined some of the work that went into making it happen.

## The requirements

We had a few fairly challenging requirements for OpenShift during this demo.  The main requirements
this blog post will discuss are:

1.  We had to be able to do a build of the 'sketchpod' application in under 30 seconds
2.  We had to be able to scale from 1 replicas of the built pod to 10 replicas in under 20 seconds
3.  We had to be able to scale from 10 -> 1026 replicas in under 3 minutes
4.  Since OpenShift v3 GA wasn't finished, we had to make the beta4 release meet all of the above

The timing requirements were dictated by the storyboard for the presentation.  I had never worked
on demo this public before.  It was a lot more like what I imagine working on a television show
would be like than working on a typical demo.

## The demo architecture

TODO: 

Let's take a look at the high-level architecture for this portion of the demo to make sure we have
a good shared understanding to base our discussion on:

TODO: block diagram here and flesh out explanation

## Initial state: a semi-primed cluster

During this post, we're going to be walking through the requirements in the order that we
investigated and tuned them while preparing for the demo.  That being the case, it makes sense
to articulate the state that the cluster was in once we really started locking down the performance
requirements we needed to hit.  The state the cluster was in once we in earnest began tuning for
performance was, at a very high level:

1.  We had functional completeness for the demo -- ie, we could run through the demo fully, and
    were basically optimizing for time
2.  We were using a cluster with 5 nodes to host the application pods, an infrastructure node, and
    a separate AWS vm for the master and datastore
3.  The cluster was created from an ansible playbook that incorporated a step of pulling the base
    image for the app onto all of the nodes in the cluster

A quick note about the third bullet.  The step that pulls the nodejs builder image onto all the
nodes in the cluster is what we'll refer to in this blog post as 'priming'.  The idea of priming
a system is pretty simple: many software systems need to do things like establish data caches or
connect to third party systems when they start up.  This means that the performance of the system
in the steady state is often different from the performance when a system is starting up.  Priming
is the activity you do when you act on a 'clean' system to make it perform like it would under
steady state.  You'll find that many of the optimizations we did took the form of some kind of
priming.

Let's take a closer look at the actual specifics of the cluster:

1.  The cluster was comprised of size AWS m3.large instances: a master/datastore host, an
    infrastructure node (router and registry), and four nodes to host the application pods
2.  The cluster was created from an ansible playbook that does the following workflow:
    1.  Spin up the ec2 instances
    2.  Create and assign the security groups
    3.  Create route53 entries for each of the hosts
    4.  Create route53 entry for the wildcard dns domain used by applications
    5.  Call the official openshift-ansible playbooks for installation and configuration
    6.  Create the demo user and project
    7.  Create the openshift-router
    8.  Create the openshift-registry
    9.  Pre-pull all needed images on the hosts
        1.  pod, builder, deployer, router, registry images for infra node
        2.  pod, builder, deployer, nodejs images for the app nodes

## Requirement 1: Build the 'sketchpod' application in under 30 seconds

TODO: needs more context probably

The first requirement we'll talk about is that in order for the demo presentation to flow
correctly, we really needed to be able to build the sketchpod image in under 30 seconds.

When I am given a requirement like the 'build in 30 seconds' one we had for this demo, the first
thing I like to do is get what's called a "p90" value around how the system performs without any
optimizations.  A p90 value is a measure in cumulative frequency analysis that measures the time
that 90% of measurements fall _under_.  In other words, if I perform some experiment X number of
times and my results indicate a p90 value of one minute, it means that 90% of the time when I
perform the experiment, it completes in _under one minute_. You can read more about p90 values
[here](https://en.wikipedia.org/wiki/Cumulative_frequency_analysis).

While performing time trials to determine the p90 value for this requirment, we noticed that the
results 'clustered' into 2 groups -- builds that took approximately 25-30 seconds and builds that
took approximately two minutes.  We expected that the timing requirement here would be easy to hit,
and so it was very surprising to see the builds taking two minutes.  

When we dug into the builds in the 'approximately 2 minutes' cluster, we found that most of the
duration of the build was spent doing a docker push to the OpenShift internal registry.  The root
cause, it turned out, was that the docker engine's checksum calculation algorithm can yield
different checksums for the same image layer bytes depending on attributes like ctime and mtime for
the layer's files on disk.  During the two-minute builds, a different checksum was being calculated
for the layers in the nodejs base image, and so the entire bytes for all the layers were being
pushed to the daemon, instead of only the bytes for the top-most layer containing the sketchpod
application code.

Luckily, we have a docker registry expert on the OpenShift team,
[Andy Goldstein](https://github.com/ncdc).  After helping us diagnose the problem, Andy built a
boutique version of the OpenShift registry that accepted new checksums for layers it already
contained and aliased them.  This allowed us to push only the topmost layer of the image we built.
Since it is important to us to fix this issues for everyone, Andy created
[an issue](https://github.com/docker/docker/issues/14018)
in the upstream docker project to share this knowledge and help us drive a fix in the upstream
registry.  

In conjunction with the modified registry, we added a priming step to the Ansible playbook that:

1.  Creates a project namespace for priming the registry
2.  Tags the nodejs builder image on each of the nodes
3.  Push the tagged nodejs builder image to the registry from each node

## Requirement 2: Resize from 1 to 10 replicas in under 20 seconds

TODO: not sure what to say here -- if I remember correctly, this was pretty much attainable with
a smaller cluster once we had the priming working correctly. 

## Requirement 3: Scale from 10 to 1026 replicas in under 3 minutes

TODO: talk about effect of cluster size and local ssd on scale-up time

## Running the demo at scale

TODO: discuss soak testing and proxy work
