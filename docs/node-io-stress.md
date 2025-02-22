---
id: node-io-stress
title: Node IO Stress Experiment Details
sidebar_label: Node IO Stress
---
------

## Experiment Metadata

<table>
  <tr>
    <th> Type </th>
    <th> Description </th>
    <th> Tested K8s Platform </th>
  </tr>
  <tr>
    <td> Generic </td>
    <td> Give IO Disk Stress on the Kubernetes Node </td>
    <td> GKE, EKS, Minikube, AKS </td>
  </tr>
</table>

## Prerequisites

- Ensure that Kubernetes Version > 1.15
- Ensure that the Litmus Chaos Operator is running by executing `kubectl get pods` in operator namespace (typically, `litmus`). If not, install from [here](https://docs.litmuschaos.io/docs/getstarted/#install-litmus)
- Ensure that the `node-io-stress` experiment resource is available in the cluster  by executing                         `kubectl get chaosexperiments` in the desired namespace. If not, install from [here](https://hub.litmuschaos.io/api/chaos/master?file=charts/generic/node-io-stress/experiment.yaml)

## Entry Criteria

- Application pods are healthy on the respective Nodes before chaos injection

## Exit Criteria

- Application pods may or may not be healthy post chaos injection

## Details

- This experiment causes io stress on the Kubernetes node. The experiment aims to verify the resiliency of applications that share this disk resource for ephemeral or persistent storage purposes.
- The amount of io stress can be either specifed as the size in percentage of the total free space on the file system or simply in Gigabytes(GB). When provided both it will execute with the utilization percentage specified and non of them are provided it will execute with default value of 10%.
- Tests application resiliency upon replica evictions caused due IO stress on the available Disk space.

## Integrations

- Node IO Stress can be injected using the chaos library: `litmus`
- This can be provided under under `LIB` variable

## Steps to Execute the Chaos Experiment

- This Chaos Experiment can be triggered by creating a ChaosEngine resource on the cluster. To understand the values to provide in a ChaosEngine specification, refer [Getting Started](getstarted.md/#prepare-chaosengine)

- Follow the steps in the sections below to create the <code>chaosServiceAccount</code>, prepare the ChaosEngine & execute the experiment.

### Prepare chaosServiceAccount

- Use this sample RBAC manifest to create a <code>chaosServiceAccount</code> in the desired (app) namespace. This example consists of the minimum necessary role permissions to execute the experiment.

#### Sample Rbac Manifest

[embedmd]:# (https://raw.githubusercontent.com/litmuschaos/chaos-charts/master/charts/generic/node-io-stress/rbac.yaml yaml)
```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: node-io-stress-sa
  namespace: default
  labels:
    name: node-io-stress-sa
    app.kubernetes.io/part-of: litmus
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-io-stress-sa
  labels:
    name: node-io-stress-sa
    app.kubernetes.io/part-of: litmus
rules:
- apiGroups: [""]
  resources: ["pods","events"]
  verbs: ["create","list","get","patch","update","delete","deletecollection"]
- apiGroups: [""]
  resources: ["pods/exec","pods/log"]
  verbs: ["create","list","get"]
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["create","list","get","delete","deletecollection"]
- apiGroups: ["litmuschaos.io"]
  resources: ["chaosengines","chaosexperiments","chaosresults"]
  verbs: ["create","list","get","patch","update"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get","list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-io-stress-sa
  labels:
    name: node-io-stress-sa
    app.kubernetes.io/part-of: litmus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: node-io-stress-sa
subjects:
- kind: ServiceAccount
  name: node-io-stress-sa
  namespace: default
```

***Note:*** In case of restricted systems/setup, create a PodSecurityPolicy(psp) with the required permissions. The `chaosServiceAccount` can subscribe to work around the respective limitations. An example of a standard psp that can be used for litmus chaos experiments can be found [here](https://docs.litmuschaos.io/docs/next/litmus-psp/).

### Prepare ChaosEngine

- Provide the application info in `spec.appinfo`. It is an optional parameter for infra level experiment.
- Provide the auxiliary applications info (ns & labels) in `spec.auxiliaryAppInfo`
- Override the experiment tunables if desired in `experiments.spec.components.env`
- To understand the values to provided in a ChaosEngine specification, refer [ChaosEngine Concepts](chaosengine-concepts.md)

#### Supported Experiment Tunables

<table>
  <tr>
    <th> Variables </th>
    <th> Description  </th>
    <th> Specify in ChaosEngine </th>
    <th> Notes </th>
  </tr>
    <tr>
    <td> TARGET_NODES </td>
    <td> Comma separated list of nodes, subjected to node io stress</td>
    <td> Mandatory  </td>
    <td> </td>
  </tr>
  <tr>
    <td> NODE_LABEL </td>
    <td> It contains node label, which will be used to filter the target nodes if TARGET_NODES ENV is not set </td>
    <td> Optional </td>
    <td> </td>
  </tr>
  <tr>
    <td> TOTAL_CHAOS_DURATION </td>
    <td> The time duration for chaos (seconds) </td>
    <td> Optional </td>
    <td> Default to 120 </td>
  </tr>
  <tr>
    <td> FILESYSTEM_UTILIZATION_PERCENTAGE </td>
    <td> Specify the size as percentage of free space on the file system  </td>
    <td> Optional  </td>
    <td> Default to 10%</td>
  </tr>
  <tr>
    <td> FILESYSTEM_UTILIZATION_BYTES </td>
    <td> Specify the size in GigaBytes(GB).  <code>FILESYSTEM_UTILIZATION_PERCENTAGE</code> & <code>FILESYSTEM_UTILIZATION_BYTES</code> are mutually exclusive. If both are provided, <code>FILESYSTEM_UTILIZATION_PERCENTAGE</code> is prioritized. </td>
    <td> Optional  </td>
    <td>  </td>
  </tr>
  <tr>
    <td> CPU </td>
    <td> Number of core of CPU to be used </td>
    <td> Optional  </td>
    <td> Default to 1 </td>
  </tr>    
  <tr>
    <td> NUMBER_OF_WORKERS </td>
    <td> It is the number of IO workers involved in IO disk stress </td>
    <td> Optional  </td>
    <td> Default to 4 </td>
  </tr> 
  <tr>
    <td> VM_WORKERS </td>
    <td> It is the number vm workers involved in IO disk stress </td>
    <td> Optional  </td>
    <td> Default to 1 </td>
  </tr>     
   <tr>
    <td> LIB  </td>
    <td> The chaos lib used to inject the chaos </td>
    <td> Optional </td>
    <td> Default to <code>litmus</code> </td>
  </tr>
   <tr>
    <td> LIB_IMAGE  </td>
    <td> Image used to run the stress command </td>
    <td> Optional  </td>
    <td> Default to <code>litmuschaos/go-runner:latest<code> </td>
  </tr>
  <tr>
    <td> RAMP_TIME </td>
    <td> Period to wait before and after injection of chaos in sec </td>
    <td> Optional  </td>
    <td> </td>
  </tr>
   <tr>
    <td> NODES_AFFECTED_PERC </td>
    <td> The Percentage of total nodes to target  </td>
    <td> Optional </td>
    <td> Defaults to 0 (corresponds to 1 node), provide numeric value only </td>
  </tr> 
   <tr>
    <td> SEQUENCE </td>
    <td> It defines sequence of chaos execution for multiple target nodes </td>
    <td> Optional </td>
    <td> Default value: parallel. Supported: serial, parallel </td>
  </tr>
  <tr>
    <td> INSTANCE_ID </td>
    <td> A user-defined string that holds metadata/info about current run/instance of chaos. Ex: 04-05-2020-9-00. This string is appended as suffix in the chaosresult CR name.</td>
    <td> Optional </td>
    <td> Ensure that the overall length of the chaosresult CR is still < 64 characters </td>
  </tr>

</table>

#### Sample ChaosEngine Manifest

[embedmd]:# (https://raw.githubusercontent.com/litmuschaos/chaos-charts/master/charts/generic/node-io-stress/engine.yaml yaml)
```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: nginx-chaos
  namespace: default
spec:
  # It can be active/stop
  engineState: 'active'
  #ex. values: ns1:name=percona,ns2:run=nginx 
  auxiliaryAppInfo: ''
  chaosServiceAccount: node-io-stress-sa
  # It can be delete/retain
  jobCleanUpPolicy: 'delete'
  experiments:
    - name: node-io-stress
      spec:
        components:
          env:
            # set chaos duration (in sec) as desired
            - name: TOTAL_CHAOS_DURATION
              value: '120'

            ## specify the size as percentage of free space on the file system
            - name: FILESYSTEM_UTILIZATION_PERCENTAGE
              value: '10'

            ## Number of core of CPU
            - name: CPU
              value: '1'

            ## Total number of workers default value is 4
            - name: NUMBER_OF_WORKERS
              value: '4'                               
            
             ## enter the comma separated target nodes name
            - name: TARGET_NODES
              value: ''
```

### Create the ChaosEngine Resource

- Create the ChaosEngine manifest prepared in the previous step to trigger the Chaos.

  `kubectl apply -f chaosengine.yml`

- If the chaos experiment is not executed, refer to the [troubleshooting](https://docs.litmuschaos.io/docs/faq-troubleshooting/) 
  section to identify the root cause and fix the issues.

### Watch Chaos progress

- View the status of the pods as they are subjected to IO disk stress. 

  `watch -n 1 kubectl get pods -n <application-namespace>`

- Monitor the capacity filled up on the host filesystem

  `watch du -h`
  
### Abort/Restart the Chaos Experiment

- To stop the pod-io-stress experiment immediately, either delete the ChaosEngine resource or execute the following command:

  `kubectl patch chaosengine <chaosengine-name> -n <namespace> --type merge --patch '{"spec":{"engineState":"stop"}}'`

- To restart the experiment, either re-apply the ChaosEngine YAML or execute the following command:

  `kubectl patch chaosengine <chaosengine-name> -n <namespace> --type merge --patch '{"spec":{"engineState":"active"}}'`

### Check Chaos Experiment Result

- Check whether the application is resilient to the io stress, once the experiment (job) is completed. The ChaosResult resource name is derived like this: `<ChaosEngine-Name>-<ChaosExperiment-Name>`.

  `kubectl describe chaosresult nginx-chaos-node-io-stress -n <application-namespace>`

## Node IO Stress Experiment Demo

- The Demo Video will be Added soon.
