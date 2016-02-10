<!--
{
"name" : "services-routing",
"version" : "1.0.0",
"title" : "Services Routing",
"description" : "",
"freshnessDate" : 2016-02-10
}
-->

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Services](#services)
  - [Creating a Service](#creating-a-service)
  - [Examine the Service](#examine-the-service)
  - [Add Pods to the Service](#add-pods-to-the-service)
- [Routing](#routing)
  - [Creating a Wildcard Certificate](#creating-a-wildcard-certificate)
  - [The Router Service Account](#the-router-service-account)
  - [Creating the Router](#creating-the-router)
  - [Viewing Router Stats](#viewing-router-stats)
  - [Exposing a Route](#exposing-a-route)
- [The Complete Pod-Service-Route](#the-complete-pod-service-route)
  - [Creating the Definition](#creating-the-definition)
  - [Project Status](#project-status)
  - [Verifying the Service](#verifying-the-service)
  - [Verifying the Routing](#verifying-the-routing)
  - [The Router Status Page](#the-router-status-page)
  - [The Web Console](#the-web-console)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Services
From the [Kubernetes
documentation](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/services.md):

    A Kubernetes service is an abstraction which defines a logical set of pods and a
    policy by which to access them - sometimes called a micro-service. The goal of
    services is to provide a bridge for non-Kubernetes-native applications to access
    backends without the need to write code that is specific to Kubernetes. A
    service offers clients an IP and port pair which, when accessed, redirects to
    the appropriate backends. The set of pods targetted is determined by a label
    selector.

From the [OpenShift
documentation](https://docs.openshift.com/enterprise/3.0/architecture/core_concepts/pods_and_services.html#services):

    A Kubernetes service serves as an internal load balancer. It identifies a set of
    replicated pods in order to proxy the connections it receives to them. Backing
    pods can be added to or removed from a service arbitrarily while the service
    remains consistently available, enabling anything that depends on the service to
    refer to it at a consistent internal address.

If you think back to the simple pod we created earlier, there was a "label":

      "labels": {
        "name": "hello-openshift"
      },

Now, let's look at a *service* definition:

    {
      "kind": "Service",
      "apiVersion": "v1",
      "metadata": {
        "name": "hello-service"
      },
      "spec": {
        "selector": {
          "name":"hello-openshift"
        },
        "ports": [
          {
            "protocol": "TCP",
            "port": 8888,
            "targetPort": 8080
          }
        ]
      }
    }

The *service* has a `selector` element. In this case, it is a key:value pair of
`name:hello-openshift`. If you looked at the output of `oc get pods` on your
master, you saw that the `hello-openshift` pod has a label:

    name=hello-openshift

The definition of the *service* tells Kubernetes that any pods with the label
"name=hello-openshift" are associated, and should have traffic distributed
amongst them. In other words, the service provides an abstraction layer, and is
the input point to reach all of the pods. Generally speaking, pod containers
should not bind directly to ports on the host. We'll see more about this later.

To really be useful, we want to make our application accessible via a FQDN,
and that is where the routing tier comes in.

## Creating a Service
We can create a service from the command line with JSON or YAML just like we
created the pod. The `hello-service.json` file has the service definition we saw
above.

As `joe`, go ahead and create the service:

    oc create -f ~/training/content/hello-service.json

## Examine the Service
`oc describe` will usually tell us some interesting things about a resource.
Let's look at the service we just created:

    oc describe service hello-service
    Name:                   hello-service
    Labels:                 <none>
    Selector:               name=hello-openshift
    Type:                   ClusterIP
    IP:                     172.30.42.80
    Port:                   <unnamed>       8080/TCP
    Endpoints:              <none>
    Session Affinity:       None
    No events.

We see that the service was assigned an IP address. This IP address will persist
for the life of the service. This can make it very convenient for other people
to actually use our service/application within an OpenShift environment. They
don't have to keep track of our pods manually.

Right now there are no endpoints in the service -- there are no matching pods.
The view in the web console will be pretty boring. Let's fix that.

## Add Pods to the Service
We can use another test file to launch some pods again. These pods all have
the label "name:hello-openshift". When we create them, and they finally come to
life, we should see them come up in the service's endpoint list.

As `joe`:

    oc create -f ~/training/content/hello-service-pods.json

Wait a few moments and then describe your service:

    oc describe service hello-service
    Name:                   hello-service
    Labels:                 <none>
    Selector:               name=hello-openshift
    Type:                   ClusterIP
    IP:                     172.30.42.80
    Port:                   <unnamed>       8080/TCP
    Endpoints:              10.1.0.6:8080,10.1.1.3:8080,10.1.1.4:8080
    Session Affinity:       None
    No events.

Now we have several endpoints behind this service. If you look at the web
console, you'll see these 3 pods all associated with our *hello-service*.

If you do a curl of the service, you should see your application (remember to
substitute whatever service IP matches your environment):

    curl 172.30.42.80:8080
    Hello OpenShift!

This is well and good, but what if I want to access this application from
outside the OpenShift environment?

# Routing
The OpenShift routing tier is how FQDN-destined traffic enters the OpenShift
environment so that it can ultimately reach pods. In a simplification of the
process, the `openshift3/ose-haproxy-router` container we will create below is a
pre-configured instance of HAProxy as well as some of the OpenShift framework.
The OpenShift instance running in this container watches for changes to route
resources on the OpenShift master.

Here is an example route resource JSON definition:

    {
      "kind": "Route",
      "apiVersion": "v1",
      "metadata": {
        "name": "hello-openshift-route"
      },
      "spec": {
        "host": "hello-openshift.cloudapps.example.com",
        "to": {
          "kind": "Service",
          "name": "hello-openshift-service"
        },
        "tls": {
          "termination": "edge"
        }
      }
    }

When the `oc` command is used to create this route, a new instance of a route
*resource* is created inside OpenShift's data store. This route resource is
affiliated with a service.

The HAProxy/Router is watching for changes in route resources. When a new route
is detected, an HAProxy pool is created. When a change in a route is detected,
the pool is updated.

This HAProxy pool ultimately contains all pods that are in a service. Which
service? The service that corresponds to the `to` directive that you
see above.

Note that while the route definition includes a refernce to a service, HAProxy
is not sending traffic to the service. HAProxy only uses the service reference
to be able to find all the pod IPs associated with the service. It proxies
traffic directly to the endpoint on the SDN.

You'll notice that the definition above specifies TLS edge termination. This
means that the router should provide this route via HTTPS. Because we provided
no certificate info, the router will provide its default SSL certificate when
the user connects. Because this is edge termination, user connections to the
router will be SSL encrypted but the connection between the router and the pods
is unencrypted.

It is possible to utilize various TLS termination mechanisms, and more details
is provided in the router documentation:

    https://docs.openshift.com/enterprise/latest/architecture/core_concepts/routes.html#secured-routes

We'll see this edge termination in action shortly.

## Creating a Wildcard Certificate
In order to serve a valid certificate for secure access to applications in our
cloud domain, we will need to create a key and wildcard certificate that the
router will use by default for any routes that do not specify a key/cert of their
own. OpenShift supplies a command for creating a key/cert signed by the OpenShift
CA which we will use.

On the master, as `root`:

    CA=/etc/origin/master
    oadm ca create-server-cert --signer-cert=$CA/ca.crt \
          --signer-key=$CA/ca.key --signer-serial=$CA/ca.serial.txt \
          --hostnames='*.cloudapps.example.com' \
          --cert=cloudapps.crt --key=cloudapps.key

Now we need to combine `cloudapps.crt` and `cloudapps.key` with the CA into
a single PEM format file that the router needs in the next step.

    cat cloudapps.crt cloudapps.key $CA/ca.crt > cloudapps.router.pem

Make sure you remember where you put this PEM file.

## The Router Service Account
Service Accounts are a unique concept in OpenShift 3. From the
[documentation](https://docs.openshift.com/enterprise/latest/dev_guide/service_accounts.html):

    Service accounts provide a flexible way to control API access without
    sharing a regular user’s credentials.

Since the router needs a way to interact with the OpenShift API (to learn about
changes on Route objects), it needs an account, as all API requests must be
authenticated. The installer has already created a service account for us. You
can inspect it with the following, as `root`:

    oc get sa/router -n default -o yaml

You'll see something like the following:

    apiVersion: v1
    imagePullSecrets:
    - name: router-dockercfg-1p0pt
    kind: ServiceAccount
    metadata:
      creationTimestamp: 2015-11-10T22:45:29Z
      name: router
      namespace: default
      resourceVersion: "356"
      selfLink: /api/v1/namespaces/default/serviceaccounts/router
      uid: be381d4b-87fc-11e5-b2a3-525400b33d1d
    secrets:
    - name: router-token-kf6of
    - name: router-dockercfg-1p0pt

Having the account is one thing, but the account still needs permissions to do
things. This is where `SecurityContextConstraints` come in. As `root`:

    oc get scc privileged -o yaml

You should see:

    allowHostDirVolumePlugin: true
    allowHostIPC: true
    allowHostNetwork: true
    allowHostPID: true
    allowHostPorts: true
    allowPrivilegedContainer: true
    allowedCapabilities: null
    apiVersion: v1
    fsGroup:
      type: RunAsAny
    groups:
    - system:cluster-admins
    - system:nodes
    kind: SecurityContextConstraints
    metadata:
      annotations:
        kubernetes.io/description: 'privileged allows access to all privileged and host
          features and the ability to run as any user, any group, any fsGroup, and with
          any SELinux context.  WARNING: this is the most relaxed SCC and should be used
          only for cluster administration. Grant with caution.'
      creationTimestamp: 2015-11-10T22:44:29Z
      name: privileged
      resourceVersion: "367"
      selfLink: /api/v1/securitycontextconstraints/privileged
      uid: 9a8aa263-87fc-11e5-b404-525400b33d1d
    priority: null
    runAsUser:
      type: RunAsAny
    seLinuxContext:
      type: RunAsAny
    supplementalGroups:
      type: RunAsAny
    users:
    - system:serviceaccount:default:registry
    - system:serviceaccount:default:router
    - system:serviceaccount:openshift-infra:build-controller

The "privileged" security context constraint is a special one, as you can see in
the annotation. The reason we need this for the router is that the router must
bind to ports 80 and 443 on the host interface. Binding to low ports like these
requires special privileges, and OpenShift only allows those privileges to
accounts in the "privileged" context. This will make more sense in a moment.

You'll see that the "users" section contains our router service account.

## Creating the Router
The router is the entry point for all traffic destined for OpenShift
v3 services. It currently supports only HTTP(S) traffic (and "any"
TLS-enabled traffic via SNI). While it is called a "router", it is essentially a
proxy.

The `openshift3/ose-haproxy-router` container listens on the host network
interface, unlike most containers that listen only on private IPs. The router
proxies external requests for route names to the IPs of actual pods identified
by the service associated with the route.

OpenShift's admin command set enables you to deploy router pods automatically.
Perform the following comands as `root`.

Let's try to create a router:

    oadm router

    error: router could not be created; you must specify a service account with
    --service-account

Well, we just created that, and it's called `router`, so let's try again:

    oadm router --service-account=router

    error: router could not be created; you must specify a .kubeconfig file path
    containing credentials for connecting the router to the master with
    --credentials

Just about every form of communication with OpenShift components is secured by
SSL and uses various certificates and authentication methods. The installer
created a `~/.kube/config` file which contains the credentials for the user
account `system:admin` (the cluster administrator) to use to access the API.
But `oadm router` is asking us what credentials the *router* should use to
communicate with the master.

    oadm router --dry-run \
    --credentials=/etc/origin/master/openshift-router.kubeconfig \
    --service-account=router

    error: router "router" does not exist (no service)

Adding that would be enough to allow the command to proceed, but, if we want
this router to work for our environment, we need to supply the wildcard cert/key
that we created for the cloud domain. Since the `router` command will create all
of the resources in the *default* project, and the *default* project has the
`nodeSelector` for the *infra* region, the router pod will land there (in the
*infra* region).

    oadm router --replicas=1 \
    --default-cert=cloudapps.router.pem \
    --credentials='/etc/origin/master/openshift-router.kubeconfig' \
    --service-account=router

If this works, you'll see some output:

    password for stats user admin has been set to xMH727koMO
    DeploymentConfig "router" created
    Service "router" created

**Note:** You will have to reference the absolute path of the PEM file if you
did not run this command in the folder where you created it.

**Note:** You will want to keep that password handy. But you can get it by
looking at the DeploymentConfiguration later. Don't worry.

Let's check the pods as `root`:

    oc get pods

In the output, you should see the router pod status change to "running" after a
few moments (it may take up to a few minutes):

    NAME             READY     REASON    RESTARTS   AGE
    router-1-rt6qk   1/1       Running   0          20s

If you `describe` the router pod, you will see that it is running on the master:

    oc describe pod router-1-rt6qk
    Name:                           router-1-rt6qk
    Image(s):                       registry.access.redhat.com/openshift3/ose-haproxy-router:v0.6.1.0
    Host:                           ose3-master.example.com/192.168.133.2
    Labels:                         deployment=router-1,deploymentconfig=router,router=router
    Status:                         Running
    IP:                             10.1.0.4
    Replication Controllers:        router-1 (1/1 replicas created)
    Containers:
      router:
        Image:              registry.access.redhat.com/openshift3/ose-haproxy-router:v0.6.1.0
        State:              Running
          Started:          Wed, 17 Jun 2015 14:49:42 -0400
        Ready:              True
        Restart Count:      0
    Conditions:
      Type          Status
      Ready         True
    Events:
      FirstSeen     LastSeen        Count   From                                    SubobjectPath           Reason           Message
      ─────────     ────────        ─────   ────                                    ─────────────           ──────           ───────
      1m            1m              1       {kubelet ose3-master.example.com}       implicitly required container POD        Pulled          Container image "openshift3/ose-pod:v3.1.0.2" already present on machine
      1m            1m              1       {scheduler }                                                    Scheduled        Successfully assigned router-1-ed1ly to ose3-master.example.com
      1m            1m              1       {kubelet ose3-master.example.com}       implicitly required container POD        Started         Started with docker id ca280668b041
      1m            1m              1       {kubelet ose3-master.example.com}       spec.containers{router} Pulling          pulling image "openshift3/ose-haproxy-router:v3.1.0.2"
      1m            1m              1       {kubelet ose3-master.example.com}       implicitly required container POD        Created         Created with docker id ca280668b041
      22s           22s             1       {kubelet ose3-master.example.com}       spec.containers{router} Pulled           Successfully pulled image "openshift3/ose-haproxy-router:v3.1.0.2"
      21s           21s             1       {kubelet ose3-master.example.com}       spec.containers{router} Created          Created with docker id 78bf7b43e0e5
      21s           21s             1       {kubelet ose3-master.example.com}       spec.containers{router} Started          Started with docker id 78bf7b43e0e5

In the very beginning of the documentation, we indicated that a wildcard DNS
entry is required and should point at the master. When the router receives a
request for an FQDN that it knows about, it will proxy the request to a pod for
a service. But, for that FQDN request to actually reach the router, the FQDN has
to resolve to whatever the host is where the router is running. Remember, the
router is bound to ports 80 and 443 on the *host* interface. Our wildcard
DNS entry points to the public IP address of the master, and the configuration
of our *default* project ensures that the router will only ever land in the
*infra* region. Since there is only one node in the *infra* region (the master),
we know we can point the wildcard DNS entry at the master and we'll be all set.

For a true HA implementation, one would want multiple "infra" nodes and
multiple, clustered router instances. Please see the "high availability"
documentation for more information on how this can be achieved:

    https://docs.openshift.com/enterprise/latest/admin_guide/high_availability.html

## Viewing Router Stats
Haproxy provides a stats page that's visible on port 1936 of your router host.
The username is `admin` and the password was displayed to you when you created
the router. If you forgot it, you can find it by `describe`ing the deployment
configuration of the router:

    oc describe dc router

To make the stats port acessible publicly, you will need to open it on your
master:

    iptables -I OS_FIREWALL_ALLOW -p tcp -m tcp --dport 1936 -j ACCEPT

You will also want to add this rule to `/etc/sysconfig/iptables` as well to keep
it across reboots. However, don't restart the iptables service, as this would
destroy the rules that have already been created by Openshift. Use the
`iptables` command to change rules on a live system.

Feel free to not open this port if you don't want to make this accessible, or if
you only want it accessible via port fowarding, etc.

**Note**: Unlike OpenShift v2 this router is not specific to a given project, as
such it's really intended to be viewed by OpenShift admins rather than project
admins or application developers.

Ensure that port 1936 is accessible and visit:

    http://admin:$YOURPASSWORDHERE@ose3-master.example.com:1936

to view your router stats.

## Exposing a Route
If you've been following along closely, right now you have three pods that all
belong to a service. We previously asked the question "How can we make this
service accessible to users outside of OpenShift?" and the answer was via the
routing layer.

Let's go ahead and actually create a route for this service by `expose`ing it.

As `joe`, execute the following:

    oc expose service hello-service -l name=hello-openshift

Wait a couple of moments, and then look at the routes with the following:

    oc get route
    NAME            HOST/PORT                                  PATH      SERVICE         LABELS
    hello-service   hello-service.demo.cloudapps.example.com             hello-service   name=hello-openshift

The `expose` command created a route for us. The `-l name=hello-openshift` added
a label to this route resource, too. Since we didn't specify an FQDN with the
command, OpenShift programmatically generates our route with the format of:

    <service_name>-<project_name>.<cloud_domain>

In our master's config we had configured the routing subdomain as
`cloudapps.example.com`. The project we're working in is `demo` and the service
name was `hello-service`. That's how the `HOST/PORT` was chosen.

If you configured your wildcard DNS correctly, you should be able to curl this
and see your application:

    curl hello-service.demo.cloudapps.example.com
    Hello OpenShift!

Hooray!

*Note:* If you get a 503 error, wait a moment and try again. The router may not
have picked up the info quite yet.

# The Complete Pod-Service-Route
The previous steps essentially build an "application" from scratch with lots of
individual JSON components. However, we can have a single JSON `List` that
describes all of the resources we want to create.

First, let's delete everything we previously created:

    oc delete all -l name=hello-openshift

You will see something like:

    routes/hello-service
    services/hello-service
    pods/hello-openshift-1
    pods/hello-openshift-2
    pods/hello-openshift-3

Since everything we created had a specific label, we can also delete everything
with that specific label. Using labels makes it very easy to find and operate on
resources in OpenShift.

Don't forget -- the materials are in `~/training/content`.

## Creating the Definition
The following is a complete definition for a pod with a corresponding service
and a corresponding route. It also includes a deployment configuration.

    {
      "kind": "List",
      "apiVersion": "v1",
      "metadata": {
        "name": "hello-service-complete-example"
      },
      "items": [
        {
          "kind": "Service",
          "apiVersion": "v1",
          "metadata": {
            "name": "hello-openshift-service",
            "labels": {
              "name": "hello-openshift"
            }
          },
          "spec": {
            "selector": {
              "name": "hello-openshift"
            },
            "ports": [
              {
                "protocol": "TCP",
                "port": 27017,
                "targetPort": 8080
              }
            ]
          }
        },
        {
          "kind": "Route",
          "apiVersion": "v1",
          "metadata": {
            "name": "hello-openshift-route",
            "labels": {
              "name": "hello-openshift"
            }
          },
          "spec": {
            "host": "hello-openshift.cloudapps.example.com",
            "to": {
              "name": "hello-openshift-service"
            },
            "tls": {
              "termination": "edge"
            }
          }
        },
        {
          "kind": "DeploymentConfig",
          "apiVersion": "v1",
          "metadata": {
            "name": "hello-openshift",
            "labels": {
              "name": "hello-openshift"
            }
          },
          "spec": {
            "strategy": {
              "type": "Recreate",
              "resources": {}
            },
            "triggers": [
              {
                "type": "ConfigChange"
              }
            ],
            "replicas": 1,
            "selector": {
              "name": "hello-openshift"
            },
            "template": {
              "metadata": {
                "creationTimestamp": null,
                "labels": {
                  "name": "hello-openshift"
                }
              },
              "spec": {
                "containers": [
                  {
                    "name": "hello-openshift",
                    "image": "openshift/hello-openshift:v1.0.6",
                    "ports": [
                      {
                        "name": "http",
                        "containerPort": 8080,
                        "protocol": "TCP"
                      }
                    ],
                    "resources": {
                      "limits": {
                        "cpu": "10m",
                        "memory": "16Mi"
                      }
                    },
                    "terminationMessagePath": "/dev/termination-log",
                    "imagePullPolicy": "IfNotPresent",
                    "capabilities": {},
                    "securityContext": {
                      "capabilities": {},
                      "privileged": false
                    },
                    "livenessProbe": {
                      "tcpSocket": {
                        "port": 8080
                      },
                      "timeoutSeconds": 1,
                      "initialDelaySeconds": 10
                    }
                  }
                ],
                "restartPolicy": "Always",
                "dnsPolicy": "ClusterFirst",
                "serviceAccount": "",
                "nodeSelector": {
                  "region": "primary"
                }
              }
            }
          }
        }
      ]
    }

In the JSON above:

* There is a pod whose containers have the label `name=hello-openshift-label` and the nodeSelector `region=primary`
* There is a service:
  * with the id `hello-openshift-service`
  * with the selector `name=hello-openshift`
* There is a route:
  * with the FQDN `hello-openshift.cloudapps.example.com`
  * with the `spec` `to` `name=hello-openshift-service`

If we work from the route down to the pod:

* The route for `hello-openshift.cloudapps.example.com` has an HAProxy pool
* The pool is for any pods in the service whose ID is `hello-openshift-service`,
    via the `serviceName` directive of the route.
* The service `hello-openshift-service` includes every pod who has a label
    `name=hello-openshift-label`
* There is a single pod with a single container that has the label
    `name=hello-openshift-label`

If your domain is different, you will need to edit the JSON before trying to
create the objects.

You might be wondering what the `DeploymentConfig` object is. We'll get to it
soon.

**Logged in as `joe`,** go ahead and use `oc` to create everything:

    oc create -f ~/training/content/test-complete.json

You should see something like the following:

    services/hello-openshift-service
    routes/hello-openshift-route
    deploymentconfigs/hello-openshift

You can verify this with other `oc` commands:

    oc get pods

    oc get services

    oc get routes

    oc get rc

    oc get dc

## Project Status
OpenShift provides a handy tool, `oc status`, to give you a summary of
common resources existing in the current project:

    oc status

    In project OpenShift 3 Demo (demo) on server https://ose3-master.example.com:8443

    svc/hello-openshift-service - 172.30.40.240:27017 -> 8080
      dc/hello-openshift deploys docker.io/openshift/hello-openshift:v1.0.6
        #1 deployed about a minute ago - 1 pod
      exposed by route/hello-openshift-route

    To see more, use 'oc describe <resource>/<name>'.
    You can use 'oc get all' to see a list of other objects.

`oc status` does not yet show bare pods or routes. The output will be
more interesting when we get to builds and deployments.

## Verifying the Service
Services are not externally accessible without a route being defined, because
they always listen on "local" IP addresses (eg: 172.x.x.x). However, if you have
access to the OpenShift environment, you can still test a service.

    oc get services
    NAME                      LABELS                 SELECTOR               IP(S)           PORT(S)
    hello-openshift-service   name=hello-openshift   name=hello-openshift   172.30.185.26   27017/TCP

We can see that the service has been defined based on the JSON we used earlier.
If the output of `oc get pods` shows that our pod is running, we can try to
access the service:

    curl $(oc get service hello-openshift-service --template '{{.spec.portalIP}}:{{index .spec.ports 0 "port"}}')
    Hello OpenShift!

This is a good sign! It means that, if the router is working, we should be able
to access the service via the route.

You might be wondering about the `--template` syntax. The `oc` command can use
Go language templates to parse/process the output of commands. If you were to
execute the `oc get service hello-openshift-service` command with `-o yaml`, you
would see the YAML representation of the service.

The `--template` we are using refers to various objects/items in that YAML.
Namely the `portalIP` and the `port`. The Go language template documentation is
a little clunky, but you can learn more about it here:

    https://golang.org/pkg/text/template/

## Verifying the Routing
Verifying the routing is a little complicated, but not terribly so. Since we
specified that the router should land in the "infra" region, we know that its
Docker container is on the master. Log in there as `root`.

We can use `oc exec` to get a bash interactive shell inside the running
router container. The following command will do that for us, as `root`:

    oc exec -it $(oc get pods | grep router | awk '{print $1}' | head -n 1) -- /bin/bash

You are now in a bash session *inside* the container running the router. We'll
talk more about `oc exec` later. For now, just realize that OpenShift is letting
you operate an interactive process inside a running Docker container inside the
OpenShift environment.

Since we are using HAProxy as the router, we can cat the `routes.json` file:

    cat /var/lib/containers/router/routes.json

If you see some content that looks like:

    "demo/hello-openshift-service": {
      "Name": "demo/hello-openshift-service",
      "EndpointTable": [
        {
          "ID": "10.1.2.6:8080",
          "IP": "10.1.2.6",
          "Port": "8080",
          "TargetName": "hello-openshift-1-ehacs"
        }
      ],
      "ServiceAliasConfigs": {
        "demo-hello-openshift-route": {
          "Host": "hello-openshift.cloudapps.example.com",
          "Path": "",
          "TLSTermination": "edge",
          "Certificates": {
            "hello-openshift.cloudapps.example.com": {
              "ID": "demo-hello-openshift-route",
              "Contents": "",
              "PrivateKey": ""
            }
          },
          "Status": "saved"
        }
      }
    }

You know that "it" worked -- the router watcher detected the creation of the
route in OpenShift and added the corresponding configuration to HAProxy.

Go ahead and `exit` from the container.

    [root@ose3-master conf]# exit

As `joe`, you can reach the route securely and check that it is using the right
certificate:

    curl --cacert /etc/origin/master/ca.crt \
             https://hello-openshift.cloudapps.example.com
    Hello OpenShift!

And:

    openssl s_client -connect hello.cloudapps.example.com:443 \
                       -CAfile /etc/origin/master/ca.crt
    CONNECTED(00000003)
    depth=1 CN = openshift-signer@1430768237
    verify return:1
    depth=0 CN = *.cloudapps.example.com
    verify return:1
    [...]

*Note:* Ctrl+C will break out of this command.

Since we used OpenShift's CA to create the wildcard SSL certificate, and since
that CA is not "installed" in our system, we need to point our tools at that CA
certificate in order to validate the SSL certificate presented to us by the
router. With a CA or all certificates signed by a trusted authority, it would
not be necessary to specify the CA everywhere.

## The Router Status Page
If you are interested, and you exposed the router admin port, you can visit the
status page and see that you will have some session information for your route.

## The Web Console
Take a moment to look in the web console to see if you can find everything that
was just created.
