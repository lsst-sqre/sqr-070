:tocdepth: 1

.. sectnum::

.. Metadata such as the title, authors, and description are set in metadata.yaml

.. TODO: Delete the note below before merging new content to the main branch.

.. note::

   **This technote is a work-in-progress.**

Abstract
========

The Influx-supplied telegraf-operator is a good concept, but it doesn't work very well for Rubin's needs.  

While we like the idea of specifying Prometheus metrics endpoints to
scrape via annotations, doing it at the pod level does not work well for
us.  Since Phalanx has the concept of applications, which generally map
one-to-one to namespaces, we would prefer an annotation system where the
annotations specifying metrics endpoints were attached to the
application object (or the namespace) rather than pod objects.  This
technote will propose an annotation syntax that would enable endpoint
specification that fits our use-case, and guide an implementation of an
operator to realize that specification.

Introduction and Problem Statement
==================================

Phalanx uses a one-app-per-namespace model for its services.  We use
telegraf at each RSP instance to scrape Prometheus metrics for those
services that support it, and telegraf-ds to collect Kubernetes metrics
for each application.

Currently, telegraf-ds is almost automated: a new application will feed
collected data to a new bucket, and because of the "bucketmaker" task
inside InfluxDB, within 15 minutes of the application having been merged
to Phalanx's ``master`` branch, a new bucket will be set up for that
application to collect its metrics.  Helm templating magic ensures that
data for the new application is correctly tagged and dispatched to the
application's bucket (the "almost" part is that this currently still
requires a resync/restart of telegraf-ds to pick up the newly-generated
configuration; recent versions of telegraf support hot reload, so this
should be a tractable problem).

However, the telegraf application is not automated.  It is necessary to
manually update the telegraf configuration with information about new
applications it should scrape Prometheus metrics for as those
applications are added to Phalanx.  This is not particularly onerous,
but it is easy to forget to do.

We are obviously not the only people to have this problem.  That is why
Influx supplies the telegraf-operator Helm chart, which provides an
operator that, if you annotate your pods (whether standalone or as part
of statefulsets or deployments), installs a sidecar telegraf container
in each pod to scrape metrics from that pod and relay them to InfluxDB
(or any other output telegraf knows about).

So why not just use the provided telegraf-operator?

In essence, I feel that it is working at the wrong layer of abstraction
for Rubin Observatory Phalanx deployments.  The scraping annotations are
put in the pod layer.  What we would prefer to do is to annotate either
the ArgoCD Application or the namespace.  Granted, that annotation will
eventually have to have specific information about a specific endpoint
to connect to, but making the annotation itself information at the
application level feels like the right choice to me.

Thus, my proposal is to write a Kubernetes Operator, possibly based on
telegraf-operator (which uses the Apache Public License), possibly *de
novo*, that consumes application-or-namespace annotations and
instantiates either a standalone telegraf application (as we have now)
with configuration generated for each prometheus endpoint specified in
the annotation, or a telegraf sidecar per target application or per
Prometheus endpoint.

Design Questions
================

There are a number of design decisions that must be made, and I would
welcome other SQuaRE team members' thoughts on them.

Annotation Location
-------------------

There are two objects we could reasonably annotate for a given
application.  These are the application Namespace (which is a core
Kubernetes object) and the Argo Application (which is provided as a CRD
when ArgoCD is installed), and which will live in the `argocd`
namespace.

My preference, I think, is to annotate the Application.  If there's some
massive change to that object, we will have much worse problems than
having to tweak our annotations.

One Telegraf?  Many Telegrafs?
------------------------------

It will reduce resource usage to have a single Telegraf application (as
we do now).  However, this means that we need to make that application
privileged in some way (or create a bunch of custom network policy
exceptions) to allow it to communicate across multiple namespaces.

On the other hand, I'm not a fan of a sidecar container per monitored
pod: that seems like a lot of containers, and (in the default config)
each requests 64M of memory, which will add up pretty fast.

I think my preference is for a middle ground between these two: the
operator can create a telegraf deployment for each application that has
the magic annotations specifying Prometheus endpoints, in that
application's namespace.  That way we don't need to do anything funny
with privilege or network policies, and we have one telegraf metrics
scraper per application-that-needs-scraping.

Annotation Format
=================

The way the existing Telegraf application handles annotations is to
specify the annotation as a YAML literal string which itself is
formatted as YAML syntax.

Doing this, and using more or less the format we're using with our own
``telegraf``, seems to make sense.

For instance, the ``argocd`` configuration, in the context of an
Application, might look like:

.. code-block:: yaml

    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: argocd
      namespace: argocd
      annotations:
        telegraf.influxdata.com/rubin-telegraf-operator: |
        - application_controller:
            "http://argocd-application-controller-metrics.argocd.svc:8082/metrics"
        - notifications_controller:
           "http://argocd-notifications-controller-metrics.argocd.svc:9001/metrics"
        - redis:
           "http://argocd-redis-metrics.argocd.svc:9121/metrics"
        - repo_server:
           "http://argocd-repo-server-metrics.argocd.svc:8084/metrics"
        - server:
           "http://argocd-server-metrics.argocd.svc:8083/metrics"
      
      finalizers:
	- resources-finalizer.argocd.argoproj.io
    spec:
      destination:
	namespace: argocd
	server: https://kubernetes.default.svc
        ...

If we do this, we don't have to care which particular pod or deployment
is providing the metrics; we simply give a key naming a metric endpoint
whose value is the (in-cluster) endpoint to scrape.

There's a bit of an annoying decoding problem here, in that the
annotation is a string; we're treating that string as YAML, decoding it
into an in-memory object in the operator, and then generating telegraf
TOML configuration from that object.  Perhaps it would make more sense
to simply make each annotation the telegraf input TOML fragment for
those services?

Operator Implementation
=======================

I have never implemented a Kubernetes Operator before.

Strimzi-registry-operator is built in Python using ``kopf`` and I see no
reason not to do the same.  The Influx telegraf-operator is, of course,
written in Go, but I don't think there is any particular reason to
follow their lead, and at this point my Python is reasonably fluent and
my Go quite rusty.

If we do that, we will obviously not be using the telegraf-operator
codebase directly.  I don't think that's a problem.  Reading it for
overall implementation ideas and translating into Python doesn't seem to
me to add difficulty or complexity.

If there is some better framework than ``kopf`` to use, I'd like to hear
about it.

Conclusion
==========

My initial preference is for annotations, specified as strings in YAML
format, to provide a manifest of endpoints to scrape for each
instrumented endpoint.

The new operator, written in Python and implemented with the ``kopf``
framework, will interpret that annotation and build a deployment of
``telegraf`` within each annotated namespace.  This deployment will build a
list of ``telegraf`` inputs and attach standardized boilerplate for
global values and an InfluxDBv2 output at monitoring.lsst.codes.  (The
output will also be tagged and sent to the correct application bucket,
so there will be some templating required for the output as well as the
inputs.)

Criticism and design argument are welcome.  So too is the question of
whether this is too much development complexity for too little gain:
after all, it's not hard to update the list of endpoints in the existing
telegraf application when a new prometheus-enabled service is added.
