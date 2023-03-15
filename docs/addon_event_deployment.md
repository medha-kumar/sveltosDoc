---
title: Event Driven Addon Distribution - Project Sveltos
description: Sveltos is an application designed to manage hundreds of clusters by providing declarative APIs to deploy Kubernetes add-ons across multiple clusters.
tags:
    - Kubernetes
    - add-ons
    - helm
    - clusterapi
    - multi-tenancy
    - Sveltos
    - event driven
authors:
    - Gianluca Mardente
---

Sveltos supports an event-driven workflow:

1. define what an event is;
2. select on which clusters watch for such events;
3. define which add-ons to deploy when event happens.

## Event definition

An _Event_ is a specific operation in the context of k8s objects.  To define an event, use the
[EventSource](https://github.com/projectsveltos/libsveltos/blob/dev/api/v1alpha1/eventsource_type.go) CRD.

Following EventSource instance define an __event__ as a creation/deletion of a Service with label *sveltos: fv*.

```yaml
apiVersion: lib.projectsveltos.io/v1alpha1
kind: EventSource
metadata:
 name: sveltos-service
spec:
 collectResources: true
 group: ""
 version: "v1"
 kind: "Service"
 labelsFilters:
 - key: sveltos
   operation: Equal
   value: fv
```

Sveltos supports custom events written in [Lua](https://www.lua.org/). 
Following EventSource instance again defines an Event as the creation/deletion of a Service with label *sveltos: fv* but using a Lua script. 

```yaml
apiVersion: lib.projectsveltos.io/v1alpha1
kind: EventSource
metadata:
 name: sveltos-service
spec:
 collectResources: true
 group: ""
 version: "v1"
 kind: "Service"
 script: |
  function evaluate()
    hs = {}
    hs.matching = false
    hs.message = ""
    if obj.metadata.labels ~= nil then
      for key, value in pairs(obj.metadata.labels) do
        if key == "sveltos" then
          if value == "fv" then
            hs.matching = true
          end
        end
      end
    end
    return hs
  end
```

In general, script is a customizable way to define complex events easily. Use it when filtering resources using labels is not enough.

When providing Sveltos with a [Lua script](https://www.lua.org/), Sveltos expects following format:

1. must contain a function ```function evaluate()```. This is the function that is directly invoked and passed a Kubernetes resource (inside the function ```obj``` represents the passed in Kubernetes resource). Any field of the obj can be accessed, for instance *obj.metadata.labels* to access labels;
2. must return a Lua table with following fields:

      - ```matching```: is a bool indicating whether the resource matches the EventSource instance;
      - ```message```: this is a string that can be set and Sveltos will print if set.

When writing an EventSource with Lua, it might be handy to validate it before using it.
In order to do so, clone [sveltos-agent](https://github.com/projectsveltos/sveltos-agent) repo.
Then in *pkg/evaluation/events* directory, create a directory for your resource if one does not exist already. If a directory already exists, create a subdirectory. Inside it, create:

1. file named ```eventsource.yaml``` containing the EventSource instance with Lua script;
2. file named ```matching.yaml``` containing a Kubernetes resource supposed to be a match for the Lua script created in #1 (this is optional);
3. file named ```non-matching.yaml``` containing a Kubernetes resource supposed to not be a match for the Lua script created in #1 (this is optional);
4. *make test*

That will load the Lua script, pass it the matching (if available) and non-matching (if available) resources and verify result (hs.matching set to true for matching resource, hs.matching set to false for the non matching resource).


## Define the add-ons to deploy

[EventBasedAddOn](https://github.com/projectsveltos/libsveltos/blob/dev/api/v1alpha1/eventbasedaddon_type.go) is the CRD introduced to define what add-ons to deploy when an event happens.

Each EventBasedAddon instance: 

1. references an [EventSource](addon_event_deployment.md#event-definition) (which defines what the event is);
2. has a clusterSelector selecting one or more managed clusters;
3. contains list of add-ons to deploy (either referencing ConfigMaps/Secrets or Helm charts).

For instance following EventBasedAddOn references the eventSource *sveltos-service* defined above.
It referenced a ConfigMap that contains a *NetworkPolicy* expressed as a template.

```yaml
apiVersion: lib.projectsveltos.io/v1alpha1
kind: EventBasedAddOn
metadata:
 name: service-network-policy
spec:
 clusterSelector: env=fv
 eventSourceName: sveltos-service
 oneForEvent: true
 policyRefs:
 - name: network-policy
   namespace: default
   kind: ConfigMap
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: network-policy
  namespace: default
data:
  networkpolicy.yaml: |
    kind: NetworkPolicy
    apiVersion: networking.k8s.io/v1
    metadata:
      name: front-{{ .Resource.metadata.name }}
      namespace: {{ .Resource.metadata.namespace }}
    spec:
      podSelector:
        matchLabels:
          {{ range $key, $value := .Resource.spec.selector }}
          {{ $key }}: {{ $value }}
          {{ end }}
      ingress:
        - from:
          - podSelector:
              matchLabels:
                app: internal
          ports:
            {{ range $port := .Resource.spec.ports }}
            - port: {{ $port.port }}
            {{ end }}
```

In above ConfigMap, __Resource__ is the Kubernetes resource in the managed cluster matching EventSource (Service instance with label sveltos:fv in this example).

Anytime a *Service* with label *sveltos:fv* is created in a managed cluster matching clusterSelector, a *NetworkPolicy* is created in the managed cluster opening ingress traffic to that service from any pod with labels *app: internal*.

For instance, if following *Service* is created in a managed cluster:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  labels:
    sveltos: fv
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

A NetworkPolicy instance is instantiated from the ConfigMap content, using information from Service (labels and ports) and it is created in the managed cluster.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  annotations:
    projectsveltos.io/hash: sha256:8e7e0a7848eef3f75aed25d1136631dd58bdb9761709a9c46153bb5d04d69e8b
  creationTimestamp: "2023-03-14T16:01:44Z"
  generation: 1
  labels:
    projectsveltos.io/reference-kind: ConfigMap
    projectsveltos.io/reference-name: sveltos-evykjze69n3bz3gavzw4
    projectsveltos.io/reference-namespace: projectsveltos
  name: front-my-service
  namespace: default
  ownerReferences:
  - apiVersion: config.projectsveltos.io/v1alpha1
    kind: ClusterProfile
    name: sveltos-8ric1wghsf04cu8i1387
    uid: ca908a7b-e9a7-457b-a077-81400b59902f
  resourceVersion: "2312"
  uid: 410e8da6-dddc-4c34-9045-8c3967119ae9
spec:
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: internal
    ports:
    - port: 80
      protocol: TCP
  podSelector:
    matchLabels:
      app.kubernetes.io/name: MyApp
  policyTypes:
  - Ingress
status: {}
```

![Event driven add-ons deployment in action](assets/event_driven_framework.gif)

This is achieved with following flow:

1. sveltos-agent in the managed cluster consumes EventSource instances and detects when an event happens;
2. when event happens, event is reported to management cluster (along with resources, since EventSource *Spec.CollectResources* is set to true) in the form of __EventReport__;
3. event-manager pod running in the management cluster, consumes the EventReport and:
      - creates a new ConfigMap in the *projectsveltos* namespace, whose content is derived from ConfigMap the EventBasedAddOn instance references, and instantiated using information coming the resource in the managed cluster (Service instance with label sveltos:fv);
      - creates a ClusterProfile.

### EventSource CollectResources setting

EventSource *collectResources* field (false by default) indicates whether any resource matching the EventSource should be collected and sent to management cluster (where it will be used to instantiate add-on templates).

If collectResources is set to true, then add-on templates can refer __Resource__ which represents kubernetes resource in the managed cluster matching EventSource (Service instance with labels sveltos: fv).

If collectResources is set to false, then add-on templates can refer __MatchingResources__ which is a corev1.ObjectReference representing resource in the managed cluster matching EventSource (Service instance with labels sveltos: fv).

In above example, there is following EventReport instance in the management cluster

```yaml
apiVersion: lib.projectsveltos.io/v1alpha1
  kind: EventReport
  metadata:
    creationTimestamp: "2023-03-14T15:55:23Z"
    generation: 2
    labels:
      eventreport.projectsveltos.io/cluster-name: sveltos-management-workload
      eventreport.projectsveltos.io/cluster-type: capi
      projectsveltos.io/eventsource-name: sveltos-service
    name: capi--sveltos-service--sveltos-management-workload
    namespace: default
    resourceVersion: "7151"
    uid: 0b71c54c-7c0e-4478-b48e-0081e2432c58
  spec:
    clusterName: sveltos-management-workload
    clusterNamespace: default
    clusterType: Capi
    eventSourceName: sveltos-service
    matchingResources:
    - apiVersion: v1
      kind: Service
      name: my-service
      namespace: default
    resources: eyJhcGlWZXJzaW9uIjoidjEiLCJraW5kIjoiU2VydmljZSIsIm1ldGFkYXRhIjp7ImFubm90YXRpb25zIjp7Imt1YmVjdGwua3ViZXJuZXRlcy5pby9sYXN0LWFwcGxpZWQtY29uZmlndXJhdGlvbiI6IntcImFwaVZlcnNpb25cIjpcInYxXCIsXCJraW5kXCI6XCJTZXJ2aWNlXCIsXCJtZXRhZGF0YVwiOntcImFubm90YXRpb25zXCI6e30sXCJsYWJlbHNcIjp7XCJzdmVsdG9zXCI6XCJmdlwifSxcIm5hbWVcIjpcIm15LXNlcnZpY2VcIixcIm5hbWVzcGFjZVwiOlwiZGVmYXVsdFwifSxcInNwZWNcIjp7XCJwb3J0c1wiOlt7XCJwb3J0XCI6ODAsXCJwcm90b2NvbFwiOlwiVENQXCIsXCJ0YXJnZXRQb3J0XCI6OTM3Nn1dLFwic2VsZWN0b3JcIjp7XCJhcHAua3ViZXJuZXRlcy5pby9uYW1lXCI6XCJNeUFwcFwifX19XG4ifSwiY3JlYXRpb25UaW1lc3RhbXAiOiIyMDIzLTAzLTE0VDE2OjAxOjE0WiIsImxhYmVscyI6eyJzdmVsdG9zIjoiZnYifSwibWFuYWdlZEZpZWxkcyI6W3siYXBpVmVyc2lvbiI6InYxIiwiZmllbGRzVHlwZSI6IkZpZWxkc1YxIiwiZmllbGRzVjEiOnsiZjptZXRhZGF0YSI6eyJmOmFubm90YXRpb25zIjp7Ii4iOnt9LCJmOmt1YmVjdGwua3ViZXJuZXRlcy5pby9sYXN0LWFwcGxpZWQtY29uZmlndXJhdGlvbiI6e319LCJmOmxhYmVscyI6eyIuIjp7fSwiZjpzdmVsdG9zIjp7fX19LCJmOnNwZWMiOnsiZjppbnRlcm5hbFRyYWZmaWNQb2xpY3kiOnt9LCJmOnBvcnRzIjp7Ii4iOnt9LCJrOntcInBvcnRcIjo4MCxcInByb3RvY29sXCI6XCJUQ1BcIn0iOnsiLiI6e30sImY6cG9ydCI6e30sImY6cHJvdG9jb2wiOnt9LCJmOnRhcmdldFBvcnQiOnt9fX0sImY6c2VsZWN0b3IiOnt9LCJmOnNlc3Npb25BZmZpbml0eSI6e30sImY6dHlwZSI6e319fSwibWFuYWdlciI6Imt1YmVjdGwtY2xpZW50LXNpZGUtYXBwbHkiLCJvcGVyYXRpb24iOiJVcGRhdGUiLCJ0aW1lIjoiMjAyMy0wMy0xNFQxNjowMToxNFoifV0sIm5hbWUiOiJteS1zZXJ2aWNlIiwibmFtZXNwYWNlIjoiZGVmYXVsdCIsInJlc291cmNlVmVyc2lvbiI6IjIyNTIiLCJ1aWQiOiIzNDg2ODE1Yi1kZjk1LTRhMzAtYjBjMi01MGFlOGEyNmI4ZWIifSwic3BlYyI6eyJjbHVzdGVySVAiOiIxMC4yMjUuMTY2LjExMyIsImNsdXN0ZXJJUHMiOlsiMTAuMjI1LjE2Ni4xMTMiXSwiaW50ZXJuYWxUcmFmZmljUG9saWN5IjoiQ2x1c3RlciIsImlwRmFtaWxpZXMiOlsiSVB2NCJdLCJpcEZhbWlseVBvbGljeSI6IlNpbmdsZVN0YWNrIiwicG9ydHMiOlt7InBvcnQiOjgwLCJwcm90b2NvbCI6IlRDUCIsInRhcmdldFBvcnQiOjkzNzZ9XSwic2VsZWN0b3IiOnsiYXBwLmt1YmVybmV0ZXMuaW8vbmFtZSI6Ik15QXBwIn0sInNlc3Npb25BZmZpbml0eSI6Ik5vbmUiLCJ0eXBlIjoiQ2x1c3RlcklQIn0sInN0YXR1cyI6eyJsb2FkQmFsYW5jZXIiOnt9fX0KLS0t
```

where resources is simply base64 encoded representation of the Service.

### EventBasedAddOn OneForEvent setting

EventBasedAddOn OneForEvent (true by default) field indicates whether to create one ClusterProfile for Kubernetes resource matching the referenced EventSource, or one for all resources.

In above example, if we create another Service in the managed cluster with label *sveltos: fv*

```bash
kubectl get services -A --selector=sveltos=fv   
NAMESPACE   NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
default     another-service   ClusterIP   10.225.134.41    <none>        443/TCP   24m
default     my-service        ClusterIP   10.225.166.113   <none>        80/TCP    52m
```

two NetworkPolicies will be created, one per Service

```bash
kubectl get networkpolicy -A
NAMESPACE   NAME                    POD-SELECTOR                          AGE
default     front-another-service   app.kubernetes.io/name=MyApp-secure   8m40s
default     front-my-service        app.kubernetes.io/name=MyApp          8m40s
```

A possible example for OneForEvent false, is when the add-ons to deploy are not template. For instance if Kyverno needs to be deployed in any managed cluster where certain event has happened.

```yaml
apiVersion: lib.projectsveltos.io/v1alpha1
kind: EventBasedAddOn
metadata:
 name: service-network-policy
spec:
 clusterSelector: env=fv
 eventSourceName: <your eventSource name>
 oneForEvent: false
 helmCharts:
 - repositoryURL:    https://kyverno.github.io/kyverno/
   repositoryName:   kyverno
   chartName:        kyverno/kyverno
   chartVersion:     v2.6.0
   releaseName:      kyverno-latest
   releaseNamespace: kyverno
   helmChartAction:  Install 
```

Currently, it is not possible to change this field.