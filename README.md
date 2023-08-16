# Adding Configurable Tolerations and Node Rules to a Workload

This guide will help you alter the convention-template used by a TAP Supply Chain to use parameters set in the Workload.yaml to configure tolerations and node affinity keys that can be used to schedule pods using these granular rules. The repo has examples of all overlays, workloads, and the result of the pods.

You can also use admission controllers and apply these same rules via Namespace annotations as outlined in the documents below:

* https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#configuration-annotation-format
* https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#podtolerationrestriction
* https://github.com/idgenchev/namespace-node-affinity

## Key Decisions

* What keys are you going to use to taint your nodes? 
* What keys are you going to use for your node affinity matching? This example is using: topology.kubernetes.io/zone

## Update the Convention Template

First you need to change the ClusterConfigTemplate convention-template to check for the tolerations and affinity parameters:

```
 #@ if hasattr(data.values.params, "tolerations"):
  tolerations:
  - key: #@ data.values.params.tolerations
    operator: "Exists"
    effect: "NoSchedule"
  #@ end
  #@ if hasattr(data.values.params, "affinity"):
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values: #@ affinity()
  #@ end
```

This can be applied via an [overlay example](convention-service-overlay.yaml) provided in the repo.

Update your tap-values on the build cluster to refer to the new overlay secret you created.

```
package_overlays:
- name: "ootb-templates"
  secrets: 
  - name: "convention-service-overlay"
```

Once these changes have commited you can then kick the sync and tap apps to pull in the new changes. 

Verify the changes are applied by looking at the convention-template.

```
k get cct convention-template
```

## Update the CNRS Config Map on Run Cluster

Knative by default doesn't allow node affinity or tolerations to be applied to clusters. You need to adjust the feature flag that allows these features. More can be found in the knative documentation:

* https://knative.dev/docs/serving/configuration/feature-flags/

You can use an overlay to alter the configmap to turn on these two feature flags:

```
apiVersion: v1
kind: Secret
metadata:
  name: config-features-overlay
  namespace: tap-install #! namespace where tap is installed
stringData:
  config-features-overlay.yaml: |
    #@ load("@ytt:overlay", "overlay")
    #@overlay/match by=overlay.subset({"kind":"ConfigMap","metadata":{"name":"config-features"}})
    ---
    data:
      #@overlay/match missing_ok=True
      kubernetes.podspec-tolerations: enabled
      #@overlay/match missing_ok=True
      kubernetes.podspec-affinity: enabled
```

Update your tap-values on the build cluster to refer to the new overlay secret you created.

```
package_overlays:
- name: "cnrs"
  secrets: 
  - name: "config-features-overlay"
```

Once these changes have commited you can then kick the sync and tap apps to pull in the new changes. 

You may need to delete the knative configmap and then kick the cnrs package to return to make the overlay work:

```
k delete configmap -n knative-serving config-features

kctrl app kick -a cnrs -n tap-install
```

## Workloads

You can now pass the keys you want to use for the tolerations and affinity rules in your pod spec via the params section of the workload:

```
spec:
  params:
    - name: tolerations
      value: test
    - name: affinity
      value: ["joe","jeremy"]
```      

This will result in a pod with this as part of the spec:

```
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - jeremy
            - joe
  tolerations:
  - effect: NoSchedule
    key: test
    operator: Exists
```


