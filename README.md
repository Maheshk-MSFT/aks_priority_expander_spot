# AKS PRIORITY EXPANDER - SPOT INSTANCES (when scaling, always start leverage spot nodes first and then dedicated if there is no spot)

https://learn.microsoft.com/en-us/azure/virtual-machine-scale-sets/spot-priority-mix?tabs=template-1

Autoscaler with priority expander would need following configurations 


1. Enable priority expander on AKS cluster  
```
az aks nodepool update --resource-group aksexpander --cluster-name aksexpander --name spotnodepool --cluster-autoscaler-profile expander=priority
```

2. Create configmap in kube-system with data & priorities including name of node pool.
```
example: apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-priority-expander
  namespace: kube-system
data:
  priorities: |-
    10:
      - aks-normalnp.*
    50:
      - aks-spotnodepool.*   
```

The priority should be a positive value. The highest value wins. For each priority value, a list of regular expressions should be given. If there are multiple node groups matching any of the regular expressions with the highest priority, one group to expand the cluster is selected each time at random. Priority values cannot be duplicated - in that case, only one of the lists will be used. If no match is found, a group will be selected at random.

3. Add toleration and node affinity to preferred spot node as preferred when scaling out
example : add following under container section
```
tolerations:
      - key: "kubernetes.azure.com/scalesetpriority"
        operator: "Equal"
        value: "spot"
        effect: "NoSchedule"
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: "kubernetes.azure.com/scalesetpriority"
                operator: In
                values:
                - "spot"
```
