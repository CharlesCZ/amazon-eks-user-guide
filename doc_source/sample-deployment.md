# Deploy a sample Linux workload<a name="sample-deployment"></a>

In this topic, you create a Kubernetes manifest and deploy it to your cluster\.

**Prerequisites**
+ You must have an existing Kubernetes cluster to deploy a sample application\. If you don't have an existing cluster, you can deploy an Amazon EKS cluster using one of the [Getting started with Amazon EKS](getting-started.md) guides\.
+ You must have `kubectl` installed on your computer\. For more information, see [Installing `kubectl`](install-kubectl.md)\.
+ `kubectl` must be configured to communicate with your cluster\. For more information, see [Create a `kubeconfig` for Amazon EKS](create-kubeconfig.md)\.

**To deploy a sample application**

1. Create a Kubernetes namespace for the sample app\.

   ```
   kubectl create namespace <my-namespace>
   ```

1. Create a Kubernetes service and deployment\. 

   1. Save the following contents to a file that's named `sample-service.yaml` on your computer\. If you're deploying to [AWS Fargate](fargate.md) pods, make sure that the value for `namespace` matches the namespace that you defined in your [AWS Fargate profile](fargate-profile.md)\. This sample deployment pulls a container image from a public repository, deploy three replicas of it to your cluster\. Then, it creates a Kubernetes service with its own IP address that can be accessed from within the cluster only\. To access the service from outside the cluster, deploy a [network load balancer](network-load-balancing.md) or [ALB Ingress Controller](alb-ingress.md)\. 

      The image is a multi\-architecture image\. If your cluster includes both x86 and Arm nodes, the pod can be scheduled on either type of hardware architecture\. Kubernetes deploys the appropriate hardware image based on the hardware type of the node it schedules the pod on\. Alternatively, if you only want the deployment to run on nodes with a specific hardware architecture, remove either `amd64` or `arm64` from the example that follows\. Or, if your cluster only contains one hardware architecture, also do the same\.

      ```
      apiVersion: v1
      kind: Service
      metadata:
        name: my-service
        namespace: my-namespace
        labels:
          app: my-app
      spec:
        selector:
          app: my-app
        ports:
          - protocol: TCP
            port: 80
            targetPort: 80
      ---
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: my-deployment
        namespace: my-namespace
        labels:
          app: my-app
      spec:
        replicas: 3
        selector:
          matchLabels:
            app: my-app
        template:
          metadata:
            labels:
              app: my-app
          spec:
            affinity:
              nodeAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  nodeSelectorTerms:
                  - matchExpressions:
                    - key: beta.kubernetes.io/arch
                      operator: In
                      values:
                      - amd64
                      - arm64
            containers:
            - name: nginx
              image: public.ecr.aws/z9d2n7e1/nginx:1.19.5
              ports:
              - containerPort: 80
      ```

      To learn more about Kubernetes [services](https://kubernetes.io/docs/concepts/services-networking/service/) and [deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/), see the Kubernetes documentation\. The containers in the sample manifest don't use network storage, but they might be able to\. For more information, see [Storage](storage.md)\. Though not implemented in this example, we recommend that you create Kubernetes service accounts for your pods, and associate them to AWS IAM accounts\. By specifying service accounts, you can have your pods have the minimum permissions that they require to interact with other services\. For more information, see [IAM roles for service accounts](iam-roles-for-service-accounts.md)

   1. Deploy the application\.

      ```
      kubectl apply -f <sample-service.yaml>
      ```

1. View all resources that exist in the `my-namespace` namespace\.

   ```
   kubectl get all -n my-namespace
   ```

   The output is as follows:

   ```
   NAME                                 READY   STATUS    RESTARTS   AGE
   pod/my-deployment-776d8f8fd8-78w66   1/1     Running   0          27m
   pod/my-deployment-776d8f8fd8-dkjfr   1/1     Running   0          27m
   pod/my-deployment-776d8f8fd8-wmqj6   1/1     Running   0          27m
   
   NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
   service/my-service   ClusterIP   10.100.190.12   <none>        80/TCP    32m
   
   NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/my-deployment   3/3     3            3           27m
   
   NAME                                       DESIRED   CURRENT   READY   AGE
   replicaset.apps/my-deployment-776d8f8fd8   3         3         3       27m
   ```

   In the output, you see the service and deployment that are specified in the sample manifest deployed in the previous step\. You also see three pods\. This is because you specified `3` for `replicas` in the sample manifest\. For more information about pods, see [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) in the Kubernetes documentation\. Kubernetes automatically creates the `replicaset` resource, even though it isn't specified in the sample manifest\. For more information about ReplicaSets, see [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) in the Kubernetes documentation\.
**Note**  
Kubernetes maintains the number of replicas that are specified in the manifest\. If this is a production deployment and you want Kubernetes to horizontally scale the number of replicas or vertically scale the compute resources for the pods, use the [Horizontal Pod Autoscaler](horizontal-pod-autoscaler.md) and the [Vertical Pod Autoscaler](vertical-pod-autoscaler.md)\.

1. View the details of the deployed service\.

   ```
   kubectl -n <my-namespace> describe service <my-service>
   ```

   The following is the abbreviated output:

   ```
   Name:              my-service
   Namespace:         my-namespace
   Labels:            app=my-app
   Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                        {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"my-app"},"name":"my-service","namespace":"my-namespace"}...
   Selector:          app=my-app
   Type:              ClusterIP
   IP:                10.100.190.12
   Port:              <unset>  80/TCP
   TargetPort:        80/TCP
   ...
   ```

   In the output, the value for `IP:` is a unique IP address that can be reached from any pod within the cluster\.

1. View the details of one of the pods that was deployed\.

   ```
   kubectl -n <my-namespace> describe pod <my-deployment-776d8f8fd8-78w66>
   ```

   The following is the abbreviated output:

   ```
   Name:         my-deployment-776d8f8fd8-78w66
   Namespace:    my-namespace
   Priority:     0
   Node:         ip-192-168-9-36.us-west-2.compute.internal/192.168.9.36
   ...
   IP:           192.168.16.57
   IPs:
     IP:           192.168.16.57
   Controlled By:  ReplicaSet/my-deployment-776d8f8fd8
   ...
   Conditions:
     Type              Status
     Initialized       True
     Ready             True
     ContainersReady   True
     PodScheduled      True
   ...
   Events:
     Type    Reason     Age    From                                                 Message
     ----    ------     ----   ----                                                 -------
     Normal  Scheduled  3m20s  default-scheduler                                    Successfully assigned my-namespace/my-deployment-776d8f8fd8-78w66 to ip-192-168-9-36.us-west-2.compute.internal
   ...
   ```

   In the output, the value for `IP:` is a unique IP that's assigned to the pod from the CIDR block, by default\. This CIDR block is assigned to the subnet that the node is in\. If you prefer that pods be assigned IP addresses from different CIDR blocks, you can change the default behavior\. For more information, see [CNI custom networking](cni-custom-network.md)\. You can also see that the Kubernetes scheduler scheduled the pod on the node with the IP address `192.168.9.36`\.

1. Run a shell on one of the pods by replacing the <value> below with a value returned for one of your pods in step 3\.

   ```
   kubectl exec -it <my-deployment-776d8f8fd8-78w66> -n <my-namespace> -- /bin/bash
   ```

1. View the DNS resolver configuration file\.

   ```
   cat /etc/resolv.conf
   ```

   The output is as follows:

   ```
   nameserver 10.100.0.10
   search my-namespace.svc.cluster.local svc.cluster.local cluster.local us-west-2.compute.internal
   options ndots:5
   ```

   In the previous output, the value for `nameserver` is the cluster's nameserver and is automatically assigned as the name server for any pod deployed to the cluster\.

1. Disconnect from the pod by typing `exit`\.

1. Remove the sample service, deployment, pods, and namespace\.

   ```
   kubectl delete namespace <my-namespace>
   ```