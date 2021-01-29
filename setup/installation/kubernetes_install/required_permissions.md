# Required permissions
All of StackState's own components can run without any extra permissions. However StackState uses Elasticsearch which has some extra requirements on the nodes it runs on. To make sure those requirements are satisfied it uses an init container that runs in privileged mode and as the root user to change the `vm.max_map_count` Linux system setting on the nodes it is started on. This init container is enabled by default, because the setting usually is lower than required and would not allow Elasticsearch to start. 

## Disabling the privileged Elasticsearch init container

In case you and/or your Kubernetes administrators don't want this behavior it can be disabled via the values.yaml used to install StackState:

```yaml
elasticsearch:
  sysctlInitContainer:
    enabled: false
```

However when disabling this you still need to ensure the `vm.max_map_count` setting is changed from its common default value of `65530` to `262144`. To inspect the current setting you can run the following command (note that it runs a privileged pod):

```
kubectl run -i --tty sysctl-check-max-map-count --privileged=true  --image=busybox --restart=Never --rm=true -- sysctl vm.max_map_count
```

If it is not at least 262144 it needs to be increased in a different way. If you don't do this Elasticsearch will fail startup and its pods will be in a restart loop. The logs will contain an error message like this:

```
bootstrap checks failed
max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
```

## Increasing Linux system settings for Elasticsearch

Depending on what your Kubernetes administrators prefer they could either set the `vm.max_map_count` to a higher default on all nodes by changing the default node configuration (for example via init scripts)  or by having a daemonset do this right after node startup. The former is very dependent on your Kuberentes cluster setup so there are no general solutions there.

However here is an example that can be used as a starting point for a daemonset to change the setting:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: set-vm-max-map-count
  labels:
    k8s-app: set-vm-max-map-count
spec:
  selector:
    matchLabels:
      name: set-vm-max-map-count
  template:
    metadata:
      labels:
        name: set-vm-max-map-count
    spec:
      # Make sure the setting always gets changed as soon as possible:
      tolerations:
      - effect: NoSchedule
        operator: Exists
      - effect: NoExecute
        key: node.kubernetes.io/not-ready
        operator: Exists
      # Optional node selector (assumes nodes for Elasticsearch are labeled `elastichsearch:yes`
      # nodeSelector: 
      #  elasticsearch: yes
      initContainers:
        - name: set-vm-max-map-count
          image: busybox
          securityContext:
            runAsUser: 0
            privileged: true
          command: ["sysctl", "-w", "vm.max_map_count=262144"]
          resources:
            limits:
              cpu: 100m
              memory: 100Mi
            requests:
              cpu: 100m
              memory: 100Mi
      # A pause container is needed to prevent a restart loop of the pods in the daemonset
      # See also this Kuberentes issue https://github.com/kubernetes/kubernetes/issues/36601
      containers:
        - name: pause
          image: google/pause
          resources:
            limits:
              cpu: 50m
              memory: 50Mi
            requests:
              cpu: 50m
              memory: 50Mi
```

To limit the number of nodes this is applied to nodes can be labeled and nodeSelectors on both this daemonset (as shown in the example) and the Elasticsearch deployment can be set to only run on nodes with the specific label. For Elasticsearch the node selector can be specified via the values:

```yaml
elasticsearch:
  nodeSelector:
    elasticsearch: yes
  sysctlInitContainer:
    enabled: false
```