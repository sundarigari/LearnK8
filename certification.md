# Curriculum
This exam curriculum includes these general domains and their weights on the exam:

| Weight| Topic                   |
|-------|-------------------------|
| 13%   | Core Concepts           |
| 18%   | Configuration           |
| 10%   | Multi-Container Pods    |
| 18%   |  Observability          |
| 20%   |  Pod Design             |
| 13%   |  Services & Networking  |
| 8%    |  State Persistence      |

# Core concepts
## install docker minikube
    docker-machine create --driver hyperv --hyperv-virtual-switch="Primary Virtual Switch" minikube
    minikube start --vm-driver="hyperv" --hyperv-virtual-switch="Primary Virtual Switch"
## Create aliases and completions
    kubectl completion bash > kc.bash
    source ./kc.bash
    
## ReplicationController and ReplicaSet
A ReplicaSet ensures that a specified number of pod replicas are running at any given time. 
However, a Deployment is a higher-level concept that manages ReplicaSets and provides declarative 
updates to Pods along with a lot of other useful features. Therefore, we recommend using 
Deployments instead of directly using ReplicaSets, unless you require custom update 
orchestration or don’t require updates at all.

This actually means that you may never need to manipulate ReplicaSet objects. Use a Deployment instead, and define your 
application in the spec section.

### Selector is the main difference between RC, RS and Deployment
RC does not need spec.selector.matchLabels. RC just needs a template inside the spec
RS and Deployment needs to select the pods from the template using a spec.selector.matchLabels manifestation  
Example replicaset using frontend.yaml 

    apiVersion: apps/v1
    kind: ReplicaSet
    metadata:
      name: frontend
      labels:
          app: guestbook
          tier: frontend
    spec:
      # modify replicas according to your case
      replicas: 3
      selector:
          matchLabels:
            tier: frontend
      template:
          metadata:
            labels:
                tier: frontend
          spec:
            containers:
              - name: php-redis
                image: gcr.io/google_samples/gb-frontend:v3
            
Note: A Deployment that configures a ReplicaSet is now the recommended way to set up replication as opposed to using 
ReplicationController. A ReplicationController ensures that a specified number of pod replicas are running at any one
time. In other words, a ReplicationController makes sure that a pod or a homogeneous set of pods is always up and 
available. ReplicaSet is apiVersion apps/v1. RS needs selector/matchLabels whereas for ReplicationController it's 
optional.  

### labels
labels are defined for k8 objects such as pods. A service or controller can use selector/matchLabels section to filter 
by key=value to select pods to control.  

### create replicaset
    kubectl create -f rs.yaml

### get replicasets
    kubectl get rs  
    kubectl get rs rsname -o yaml > rsname.yaml

### edit replicasets
    kubectl edit replicaset rsname 
    
will extract and open yaml in vi. Edit and save to apply.  Make sure the existing pods need to be deleted or the rs 
itself needs to be deleteted and recreated to have new pods with new changes

### delete replicasets
    kubectl delete replicaset rs-name  

### scale replicaset
    kubectl scale --replicas=N rsdef.yaml  
    kubectl scale rs rs-name --replicas=N

### update/replace replicaset
    kubectl replace -f rsdef.yaml  

## Deployments
Deployments are 
    apiVesrion: apps/v1  
    kind: Deployment  
    template: # has pod spec  
        spec:
            containers:
            - name:

Deployment creates a replicaset. Creates pods, perform rolling updates and rollback if needed.  

## Namespaces
namespaces have 1) permission policies 2) quota such as max number of nodes/services/deployments etc. You can access 
resources across namespaces using fully qualified name such as : objectname.namespace-name.svc.cluster.local  
You can separate Dev and Prod using two namespaces. 

First create Namespace object using  yaml.

create-ns.yaml  
 
    apiversion: v1  
    kind: Namespace  
    metadata   
        name: dev  

or simply use
    
    kubectl create namespace dev  

two ways to specify namespace while creating an object such as pod  
    
    kubectl create -f zzz.yaml --namespace dev  

or use metadata.namespace in the yaml to specify   

    metadata  
        name:   mypod  
        namespace: dev  

how to set default namespace to dev?  
 
    kubectl config  set-context $(kubectl config current-context) --namespace=dev  

ResourceQuota  

    apiversion: v1  
    kind: ResourceQuota  
    metadata  
        name: compute-quota  
        namespace: dev  
    spec:  
        hard:  
            pods: "10"  
            requests.cpu: "4"  
            requests.memory: 5Gi  
            limits.cpu: "10"  
            limits.memory: 5Gi  
### how to access db-service in dev name space from default namespace
within the same namespace, use:  

    mysql.connect("db-service")

from outside the dev namespace, use   

    mysql.connect("db-service.dev.svc.cluster.local")
    
where cluster.local is the default domain name of the cluster
svc.cluster.local is the default subdomain for all services in the cluster
![](https://imgur.com/qdDAXcl.jpg)

### how to get pods from all namespaces
    kubectl get pods --all-namespaces
    
### how to limit resources in a namesapce
When several users or teams share a cluster with a fixed number of nodes, there is a concern that one team could use more 
than its fair share of resources.

Resource quotas are a tool for administrators to address this concern. A resource quota, defined by a ResourceQuota object, 
provides constraints that limit aggregate resource consumption per namespace. It can limit the quantity of objects that 
can be created in a namespace by type, as well as the total amount of compute resources that may be consumed by 
resources in that project.    

example to create many resourcequota objects

    apiVersion: v1
    kind: List
    items:
    - apiVersion: v1
      kind: ResourceQuota
      metadata:
        name: pods-high
      spec:
        hard:
          cpu: "1000"
          memory: 200Gi
          pods: "10"
        scopeSelector:
          matchExpressions:
          - operator : In
            scopeName: PriorityClass
            values: ["high"]
    - apiVersion: v1
      kind: ResourceQuota
      metadata:
        name: pods-medium
      spec:
        hard:
          cpu: "10"
          memory: 20Gi
          pods: "10"
        scopeSelector:
          matchExpressions:
          - operator : In
            scopeName: PriorityClass
            values: ["medium"]
    - apiVersion: v1
      kind: ResourceQuota
      metadata:
        name: pods-low
      spec:
        hard:
          cpu: "5"
          memory: 10Gi
          pods: "10"
        scopeSelector:
          matchExpressions:
          - operator : In
            scopeName: PriorityClass
            values: ["low"]

# Configuration
## Docker Containers
Kubernetes containers are meant to run a task to completion and exit. Once the task is completed the container exits 
the cmd statement specified in the dockerfile. 
In the docker file there are two important items: ENTRYPOINT and CMD
Entrypoint + cmd tells docker what command to run to completion along with arguments. 

    CMD "ngnix"  
    CMD ["sleep", "5"]
    CMD sleep 5
    

entry point   

    ENTRYPOINT "sleep"  
    
with entrypoint just pass the parameters to the entrypoint (without mentioning the entrypoint itself) in the cmd  
   
    docker run my-sleper-image 20  
    
ENTRYPOINT is always prepended to CMD  

    FROM ubuntu
    ENTRYPOINT ["sleep"]
    CMD ["20"]

you can simply run using docker run. The arguments passed to docker run will replace CMD. Entrypoint will not change.

    dockr run my-image 22

You can override the entrypoint using   

    docker run --entrypoint sleepv2 ubuntu-sleeper 10
the new entrypoint will be sleepv2 and new CMD will be 10 thus making the docker sleep for 10 sec using sleepv2 

how to pass command arguments from yaml in kubernetes?  use spec.containers.args: ["arg1", "arg2"]

    spec
        containers
            -   name: ccc
                args: ["20"]

### To override the entrypoint + cmd of docker
In yaml in kubernetes, use spec.containers.command and spec.containers.args. These two are used to
override enrrypoint + cmd of the docker image.

    spec
        containers
            -   name: ccc
                command: ["sleep2.0"]
                args: ["20"]

k8 command is same as docker ENTRYPOINT. k8 args is same as docker CMD

### pass environment variables to container using env
    spec
        containers
        -   name
            env:
                -   name: accessId
                    value:  CVGHDFGJH087654ASDFGHJK
                -   name: accessKey
                    value: DFN456DF$FG23456DFGH^&C456
        
### run using command line passing env        
                    
    docker run -e accessID=XXXXXXXXXX imagename
    
in your docker image, in the entrypoint java main program (or ny other language), you can access the above environment 
variables using:

    public static void main(String[] args) {        
        String accessId = System.getenv("accessId");  
        String accessKey = System.getenv("accessKey");  
        
                    
## ConfigMaps
ConfigMaps allow you to decouple configuration artifacts from image content to keep containerized applications portable.
1) First Create a config map
2) Next inject into a pod


    kubectl create configmap mycmp --from-literal=key1=val1 --from-literal=key2=val2
using artifact yml file 

    kubectl create -f mycmp.yaml
    kubectl get configmaps  
    kubectl describe configmaps  
    
yml:

    apiVersion: v1
    kind: Configmap
    metadata:
        name: mycmp
    data:
        key1:val1
        key2:val2

### inject configMap data into pod
    use spec.containers.envFrom.configmapRef.name=nameofthe-config-map  
    use spec.containers.envFrom.configmapRef.key=nameofthe-key  

### to inject the *entire* config map into a container spec
    spec
        containers
        -   name: nm
            envFrom:
                - configMapRef:
                    name: configmap1  # entire configmap1 is sent as env
                - configMapRef:
                    name: configmap2  # entire configmap2 is sent as env

### to inject a *single value* from a config map app-config  into a container spec
    spec  
        containers  
        -   name  
            env:  
            -   name: APP_COLOR  
                valueFrom:    
                    configMapKeyRef:  
                        name: app-config  # only one key AAPP_COLOR_MAIN is sent as env var APP_COLOR
                        key: APP_COLOR_MAIN
## Secrets
    kubectl create secret generic mysec --from-literal=REDIS_PASS=password-plain-txt 
    kubectl create secret generic mysec --from-file=mysecrets.properties    
### yaml
    apiVersion: v1
    kind: Secret
    metadata:
        name: app-secrets
    data:
        REDIS_PASS: base64-encoded-pass  # use echo -n pass | base64 to get the base64-encoded-pass
        REDIS_HOST: base64-encoded-hostname # use echo -n hostname | base64 to get the base64-encoded-hostname
### base64 encoding
    echo -n redis123 | base64
    bXlwYXN3b3Jk
### base64 decoding
    echo -n bXlwYXN3b3Jk | base64  --decode
    mypassword
### Inject the *entire* secret into a container spec as env vars
    spec  
        containers  
        -   name  
            envFrom:  
                -   secretRef:   
                        name: mysecrets1  
                -   secretRef:   
                        name: mysecrets2   
                        
### Inject *entire* secret into a pod as volume and files with names equal to the secret keys
    kubectl create secret generic ssh-key-secret --from-file=ssh-privatekey=/path/to/.ssh/id_rsa --from-file=ssh-publickey=/path/to/.ssh/id_rsa.pub

Above cmd will create a secret with two keys. The value is the entire contents of the file.
Below yaml creates a secret with one key which is .secret-file and a  pod wth the  secret mounted as volume.
 
    kind: Secret
    apiVersion: v1
    metadata:
      name: dotfile-secret
    data:
      .secret-file: dmFsdWUtMg0KDQo=
    ---
    kind: Pod
    apiVersion: v1
    metadata:
      name: secret-dotfiles-pod
      labels:
        name: secret-test
    spec:
      volumes:
      - name: secret-volume
        secret:
          secretName: ssh-key-secret
      containers:
      - name: dotfile-test-container
        image: k8s.gcr.io/busybox
        command:  ["sh", "-c", "cd /etc/secret-dir && ls -la  && cat .secret-file"]
        volumeMounts:
        - name: secret-volume
          readOnly: true
          mountPath: "/etc/secret-dir"

The keys of the secret are now available as files inside mount path (/etc/secrets-dir folder)

create this pod using  

    kubectl apply -f secretWithDotFiles.yml

and see the logs using

    kubectl logs secret-dotfiles-pod
                    
### Inject a *single key* from a secrets into a container spec
    spec
        containers
        -   name
            env:
            -   name: REDIS_PASS
                valueFrom:
                    secretKeyRef:
                        name: app-secrets
                        key:  REDIS_PASS

## Docker Security
By default docker runs with root user with pid=1. This root user has lesser privileges than the host's root user. For 
example docker root user can't reboot host.

You can run docker container using a different userid 
 
    docker run --user=1000 ubuntu sleep 3600
    
Specify user in the Dockerfile
    
    FROM Ubuntu
    USER 1000

Add/Drop capabilities to docker user

    docker run --cap-add  MAC_ADMIN ubuntu 
    docker run --cap-drop KILL ubuntu 

Add all capabilities 
    
    docker run --privileged  ubuntu 


##  Security Contexts
Pod security context specifies the context for security of the pods. Container security context specifies the context for 
the container. Security context for container (spec.container.securityContext) overrides the security context for the pod 
(spec.securityContext)

### Security Context at pod level and container level using manifest yaml file
    spec
        securityContext: # pod level security context
            runAsUer: 1000
            capabilities:
                add: ["MAC_ADMIN"]
        containers
        -   name: c1
            image: Ubuntu     
            securityContext:  # container level security context. Overrides pod level sc
                runAsUer: 1000 # runasUser runasGroup fsGroup
                capabilities:
                    add: ["MAC_ADMIN"]      
        

### Set capabilities for a Container
With Linux capabilities, you can grant certain privileges to a process without granting all the privileges of the root 
user. To add or remove Linux capabilities for a Container, include the capabilities field in the securityContext section 
of the Container manifest. With Linux capabilities, you can grant certain privileges to a process without granting all 
the privileges of the root user. To add or remove Linux capabilities for a Container, include the capabilities field in 
the securityContext section of the Container manifest.
    
    
## Service accounts
A service account provides an identity for processes that run in a Pod.
When you (a human) access the cluster (for example, using kubectl), you are authenticated by the apiserver as a 
particular *User Account* (currently this is usually admin, unless your cluster administrator has customized your cluster). 
Processes in containers inside pods can also contact the apiserver. When they do, they are authenticated as a particular 
Service Account (for example, default).

RBAC can be used to a assign roles to a service account. 
Kubernetes has  
    1) user account Ex: admin, developer etc  used by humans  
    2) service account ex: Prometheus Jenkins used by application to interact with kubernetes api  

    kubectl create serviceccount dashboard-sa
    kubectl get serviceaccount

serviceaccount creates automatically a token, creates a secret object, copies the token into the secret object and then 
links the secret to the sa. To see the token issue kubectl describe secret command for the secret used by the sa.

Even if the third party application is not running inside the cluster, it can still call our kubernetes api using this token.
But if the 3rd party app is hosted inside the cluster, you can simply mount a service account's secret as a volume into the pod


serviceaccounts contain a Tokens object which is actually a secret object.
kubectl describe serviceaccount dashboard-sa

each namespace has a default sa
all pods when started, mount the default sa as a volume so that pod has access to the token for default sa
the service account secret is mounted at: /var/run/secrets/kubernetes.io/serviceaccount which contains three files: ca.crt, namespace, token


The pod spec contains serviceAccountName: saname or it will use default sa
In a deployment, specify in the pod spec under template as  serviceAccountName: saname

    apiVersion
    metadata
    kind: Deployment
    spec
        template:
            spec:
                serviceAccountName: saname  # for the entire pod
                containers:
                -
when you change the sa for a pod, you have to delete the pod and recreate it to take effect.
Incase of a deployment, no need to delete the pod. Just re apply the dep. yaml

you can choose to not mount any sa into a pod by using

    spec
        automountServiceAccountToken: false

## Resource Requirements
Min amount of requirement for a pod to run: .5 cpu and 256Mi mem

Ki = 1024 (Kibibyte)
K=1000
Mi=1048576  (Mebibyte)
M=1000,000
Gi = 1024X1024X1024 (Bibibyte)
G = 1000000000 bytes

default limit for pod: 1 cpu 512Mi

you can change the resources and limits in the spec  at the container level (unlike a serviceacount which is set at the pod level)

    spec
      containers
      - image
        resources:  
            requests:  
                memory:  
                cpu:  
            limits:  
                memory:  
                cpu:  

## taints and tolerations
a node can be tainted (using Node 1 is tainted with: taint=blue) so that not all pods can be hosted on that node.  
Then pods can be specified to tolerate a taint. For example pod A can ve specified as tolerate=blue and pod b can be specified to tolerate=red
in such scenario only pod A will be placed in node 1.  
pod B will be placed on other nodes.  

    kubectl taint nodes node-nme key=value:NoSchedule  ## NoSchedule PreferNoSchedule NoExecute  
    kubectl taint nodes node-nme app=blue:NoSchedule  ## NoSchedule PreferNoSchedule NoExecute  

in the pod spec use tolerations  

    spec:  
        tolerations:   
         -  key: "app"
            operator: "Equal"
            value: "blue"
            effect: "NoSchedule"


    kubectl describe node kubemaster | grep Taint
will show that master is tainted hence no pods are scheduled to master node  

## nodeSelector
Enables to choose a node (as opposed to taints which disables a node for pods) nodes have labels in metadata as follows:


    apiVersion: v1  
    kind: Node  
    metadata:  
    labels:  
        beta.kubernetes.io/arch: amd64  
        beta.kubernetes.io/os: linux  
        color: blue  
        kubernetes.io/hostname: node01  

How to label the node with size=Large? 

    kubectl label nodes nodename key=value  
    kubectl label nodes node01 size=Large    


in the pod definition, you can specify nodeSelector by label  

    apiVersion: v1
        kind: Pod
        metadata:
          name: nginx
          labels:
            env: test
        spec:
          containers:
          - name: nginx
            image: nginx
            imagePullPolicy: IfNotPresent
          nodeSelector:
            disktype: ssd

labels are simple mechanism to choose a pod.

## nodeAffinity and anti affinity features

Using nodeAffinity you can use complex logical operators such as In,
NotIn Exists etc. to filter based on multiple lables/values


    spec:  
        affinity:  
            nodeAffinity:
                requiredDuringSchedulingIgnoreDuringExecution:
                # or preferredDuringSchedulingIgnoreDuringExecution planned: requiredDuringSchedulingRequiredDuringExecution preferredDuringSchedulingRequiredDuringExecution
                    nodeSelectorTerms:
                        - matchingExpression:
                            -   key: size
                                operator: In
                                # NotIn Exists (any value as long as key exists) etc
                                values:
                                - Large
                                - Medium

Like tolerations, nodeSelector and  nodeAffinity are applied at pod level as a property of pod spec (not at container level)

# Multi-Container pods (10%)

patterns of MCPs:

    Ambassador: A container that can switch between dev/test/prod dB's based the environment  
    Adaptor: A log agent container may need a log agent adapter container to convert the logs to a standard format  
    Sidecar: A Webserver container may need a log agent container  

They share the same ntwork space (refer using localhost) and storage  

Sharing volumes  

    spec:  
    containers:  
    - name: app  
        image: kodekloud/event-simulator  
        volumeMounts:  
        - mountPath: /log  
        name: log-volume  

    - name: sidecar  
        image: kodekloud/filebeat-configured  
        volumeMounts:  
        - mountPath: /var/log/event-simulator/  
        name: log-volume  

    # create a volume  
    volumes:  
    - name: log-volume  
        hostPath:  
        # directory location on host  
        path: /var/log/webapp  
        # this field is optional  
        type: DirectoryOrCreate

3 pod elk stack
pod1:
    containers:
        app (writes logs to /app/logs mount pointing to log-volume)
        sidecar (filebeat reads logs from/var/log/event-simulator/ mount pointing to log-volume)
    volumes:
        - hostPath:
              path: /var/log/webapp
              type: DirectoryOrCreate
              name: log-volume

pod2:
    elastic-search : gets data from filebeat
pod3:
    kibana: dashboard for elastic search


# Observability
## Readiness Probes
Pod stages  

    Pending (untill a node is found)  
    ContainerCreating (pulls image and creates containers)  
    Running  

Pod Conditions are boolean variables (true or false). kubectl describe pod pod-name will show all the conditions with the true/flase values  

    PodScheduled  
    Initialized  
    ContainersReady  
    Ready  

Even when the pod Ready condition is true the web application or db application could still be not ready (warming up state)  

## readinessProbes (for terminated containers)
To solve this you need to add readinessProbes in the pod definition files under spec.

    spec:  
        readinessProbe:  
            httpGet:  
                path: /api/ready  
                port: 8080  
            initialDelaySeconds: 10
            periodSeconds: 5  # how often to probe
            failureThresholds: 8 # Total number of attempts. default is 3
    spec:  
        readinessProbe:  
            tcpSocket:  
                port:  
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThresholds: 8 # default is 3
    spec:  
        readinessProbe:  
            command:  
            - cat  
            - /app/is-ready  
            initialDelaySeconds: 10  
            periodSeconds: 5  
            failureThresholds: 8 # default is 3  

## livenessProbes (for frozen containers)
If a container crashes the pod will also crash and k8 will create a new pod  
But sometimes the container may not crash but may not be accisible due to a bug or going into some infinite loop  

    spec:  
        livenessProbe:  
            httpGet:    
                path: /api/ready  
                port: 8080  
            initialDelaySeconds: 10  
            periodSeconds: 5  
            failureThresholds: 8 # default is 3  

## Container logging
    docker logs -f dockername  # shows the logs for the container
    kubectl logs -f podname dockername  # shows the logs for the container dockername in the pod: podname

## Monitor resource consumption
CPU, memory  
Open Source: Prometheus, MetricsServer (HeapSter), ElasticStack
proprietary: DataDog, DynaTree   

## MetricsServer (HeapSter is deprecated)
In memory metrics aggregator. Gathers metrics from nodes, pods, summarizes them to a in-memory database
Each node run an agent called  kubelet, which is responsible for receiving instructions from kubernetes master node
and implement the instructions viz  launching/shutting down pods and containers

cAdvisor is a sub component of kubelet, which also runs on all nodes.
cAdvisor is responsible for aggregating the metrics of the node and sending to MetS

    minikube addons enable metrics-server  

to see the metrics. Only one metrics server per k8 cluster   

    kubectl top node  
    kubectl top pod  

# Pod Design
## Labels and Selectors
labels are used to tag a kubernetes object. Labels are mentioned in the metadat  

    metadata:  
        labels:  
            app: App1  
            function: Web-Server  

to list all objects with a specific label use;  
    
    kubectl get pods --selector app=App1,tier=Middle-Tier,bu=Finance  # all three labels must match  

selector selects objects from template/metadata  

example  

    apiVersion: apps/v1  
    kind: ReplicaSet  
    metadata:  
        labels:  
            app: RS-App1   # this is only used to select a RS  
        annotations:  
            version: 1.0.1   # annotations are used for commenting purpose only  
    spec:  
        selector:  
            matchLabels:  
                app: App1            # >-------- choose all objects with labels app=App1 from the below template  
        template:                    #         |  
            metadata:                #         |  
                labels:              #         |  
                    app: App1        # <--------  
            spec:  

## Rollout and Versioning in Deployments
When you create a deployment it triggers a new rollout which inturn creates a new depl. revision
Revision1 is created when a new deployment with appimage:v1 is created
when the new image(or spec) appimage:v2 applied to the deployment, then a new Revision2 is applied

    kubectl rollout status deployment/depname   

## shows the status deployment depname successfully rolled out
    kubectl rollout history deployment/depname   

## shows the history of deployment depname
    deployment.extensions/redis  
    REVISION  CHANGE-CAUSE  
    1         <none>  
    2         <none>  

changing the number of replicas does not count as new revision since we are not changing the template.  
changing image counts as new revision since we are chaning the template  

## deployment strategies
Recreate strategy: Delete all pods and re-create them with newer image. Cons: App will be unavailable for some time  
Rolling Update Strtegy: This is default. Deletes one pod and creates one new pod, then deletes another old pod and creates a new pod and so on

    strategy:   
        rollingUpdate:  
        maxSurge: 25%  
        maxUnavailable: 25%  
        type: RollingUpdate   # few will be updated at a time  

    strategy:  
        type: Recreate # all will be updated at a time. delete all create all  

    kubectl describe deployment/redis  

Events:  

    Type    Reason             Age                 From                   Message  
    ----    ------             ----                ----                   -------  
    Normal  ScalingReplicaSet  11m                 deployment-controller  Scaled up replica set redis-77fbbd56c to 1  
    Normal  ScalingReplicaSet  11m                 deployment-controller  Scaled down replica set redis-785f9d6bfb to 0  
    Normal  ScalingReplicaSet  9m12s               deployment-controller  Scaled up replica set redis-77fbbd56c to 2  
    Normal  ScalingReplicaSet  8m31s               deployment-controller  Scaled up replica set redis-77fbbd56c to 3  
    Normal  ScalingReplicaSet  92s                 deployment-controller  Scaled down replica set redis-77fbbd56c to 2  
    Normal  ScalingReplicaSet  62s (x2 over 108s)  deployment-controller  Scaled up replica set redis-77fbbd56c to 5  
    Normal  ScalingReplicaSet  62s                 deployment-controller  Scaled down replica set redis-77fbbd56c to 0  
    Normal  ScalingReplicaSet  52s                 deployment-controller  Scaled up replica set redis-785f9d6bfb to 5  
    Normal  ScalingReplicaSet  2s                  deployment-controller  Scaled up replica set redis-785f9d6bfb to 10  

## Rollback
    kubectl rollout undo deployment/redis

## kubectl run cmd actually creates a deployment  
    kubectl run ngnix --image nginx  

Summarize commands:  

Create: 
    
    kubectl create -f dep.yaml  
    
Get:   

    kubectl get deployments  
    
update:    

    kubectl apply -f dep.yaml  
    kubectl set image  deployment/depname container-name=new-image  
    
Status: 

    kubectl rollout status deployment/depname  
    kubectl rollout history deployment/depname  

Rollback:   

    kubectl rollout undo  deployment/depname

## Jobs  - Run to Completion
By deafut k8 recreates the container if it exits (ppd: spec.restartPolicy: Always) , in an effort to keep it running continuously.

A ReplicaSet is a set of pods that run continuously
A job is like a RS which is a set of pods, only differes where the pods do not run continuously ((ppd: spec.restartPolicy: Never)

A Job creates one or more Pods and ensures that a specified number of them successfully terminate. As pods successfully
complete, the Job tracks the successful completions. When a specified number of successful completions is reached, the 
task (ie, Job) is complete. Deleting a Job will clean up the Pods it created.

A simple case is to create one Job object in order to reliably run one Pod to completion. The Job object will start a 
new Pod if the first Pod fails or is deleted (for example due to a node hardware failure or a node reboot).

You can also use a Job to run multiple Pods in parallel

    apiVersion: batch/v1
    kind: Job
    metadata:
      name: pi
    spec:
      completions: 3
      # pods are created one after the other by default ( parallelism: 1). 2nd pod is created after the 1st pod completes
      # is a pod fails then its not counted. You keep doing untill the number of successful completions = 3
      parallelism: 3  # if you specify non 1 number then this number of pods run parallelly
      template:
        spec:
          containers:
          - name: pi
            image: perl
            command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
          restartPolicy: Never
      backoffLimit: 4
 

The above spec will try to run the pod if it exits thus making sure the pod nerver is always running  
Jobs do not run for ever. They perform a task such as adding two numbers and exit with the result copied to some text
 file on a log server  

Job is like replicaset, a set of pods except the pods run to completion and do not re-start  
job definition  

    apiVersion: batch/v1  
    kind: Job  
    metadata:  
        name: math-add-job  
    spec:
        backoffLimit: 20  # number of failed attempts before k8 gives up
        completions: 3  # pods are created sequentially. 2nd pod is created after 1st pod is finished. If a pod fails then it creates a new one until it has 3 successful creations  
        parallelism: 3  # run three parallel  
        template:  
            spec:  
                containers:  
                - name: math-add  
                image: ubuntu  
                command: ["expr","3", "+", "2"]  
                restartPolicy: Never  

Pod Backoff failure policy
There are situations where you want to fail a Job after some amount of retries due to a logical error in configuration etc. To do so, set .spec.backoffLimit to specify the number of retries before considering a Job as failed. The back-off limit is set by default to 6. Failed Pods associated with the Job are recreated by the Job controller with an exponential back-off delay (10s, 20s, 40s …) capped at six minutes. The back-off count is reset if no new failed Pods appear before the Job’s next status check.


### Parallel Jobs
There are three main types of task suitable to run as a Job:

### Non-parallel Jobs
normally, only one Pod is started, unless the Pod fails.
the Job is complete as soon as its Pod terminates successfully.
Parallel Jobs with a fixed completion count:
specify a non-zero positive value for .spec.completions.
the Job represents the overall task, and is complete when there is one successful Pod for each value in the range 1 
to .spec.completions.
not implemented yet: Each Pod is passed a different index in the range 1 to .spec.completions.

### Parallel Jobs with a work queue:
do not specify .spec.completions, default to .spec.parallelism.
the Pods must coordinate amongst themselves or an external service to determine what each should work on. For example, 
a Pod might fetch a batch of up to N items from the work queue.
each Pod is independently capable of determining whether or not all its peers are done, and thus that the entire Job 
is done.
when any Pod from the Job terminates with success, no new Pods are created.
once at least one Pod has terminated with success and all Pods are terminated, then the Job is completed with success.
once any Pod has exited with success, no other Pod should still be doing any work for this task or writing any output.
 They should all be in the process of exiting.
For a non-parallel Job, you can leave both .spec.completions and .spec.parallelism unset. When both are unset, both 
are defaulted to 1.

For a fixed completion count Job, you should set .spec.completions to the number of completions needed. You can set 
.spec.parallelism, or leave it unset and it will default to 1.

For a work queue Job, you must leave .spec.completions unset, and set .spec.parallelism to a non-negative integer.

## Cronjobs

Cronjob contains a Job under a jobTemplate, which contains a container under the template
Cronjob definition (notice jobTemplate and total 3 specs)


    apiVersion: batch/v1beta1  
    kind: CronJob  
    metadata:  
        name: report-cron-job  
    spec:  
        schedule: "*/1 * * * *"   
        jobTemplate:   
            spec:  
                completions: 3  
                # pods are created sequentially. 2nd pod is created after 1st pod is finished. 
                # If a pod fails then it creates a new one   until it has 3 successful creations  
                parallelism: 3  # run three parallel     
                template:  
                    spec:  
                        containers:  
                        - name: math-add  
                        image: ubuntu  
                        command: ["expr","3", "+", "2"]  
                        restartPolicy: Never  

schedule format

     ┌───────────── minute (0 - 59)
     │ ┌───────────── hour (0 - 23)
     │ │ ┌───────────── day of the month (1 - 31)
     │ │ │ ┌───────────── month (1 - 12)
     │ │ │ │ ┌───────────── day of the week (0 - 6) (Sunday to Saturday;
     │ │ │ │ │                                   7 is also Sunday on some systems)
     │ │ │ │ │
     │ │ │ │ │
     * * * * * command to execute  

## DaemonSets
### What is daemonset

A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to 
them. As nodes are removed from the cluster, those Pods are garbage collected. Deleting a DaemonSet will clean up the 
Pods it created.

Some typical uses of a DaemonSet are:

running a cluster storage daemon, such as glusterd, ceph, Apache bookkeeper on each node.
running a logs collection daemon on every node, such as fluentd or logstash.
running a node monitoring daemon on every node, such as Prometheus Node Exporter, Sysdig Agent, collectd, 
Dynatrace OneAgent, AppDynamics Agent, SignalFx Agent, Datadog agent, New Relic agent, Ganglia gmond or Instana agent.

In a simple case, one DaemonSet, covering all nodes, would be used for each type of daemon. A more complex setup might 
use multiple DaemonSets for a single type of daemon, but with different flags and/or different memory and cpu requests 
for different hardware types.
### Communicating with Daemon Pods
Some possible patterns for communicating with Pods in a DaemonSet are:

#### Push: Pods in the DaemonSet are configured to send updates to another service, such as a stats database. They do not have clients.
#### NodeIP and Known Port: Pods in the DaemonSet can use a hostPort, so that the pods are reachable via the node IPs. Clients know the list of node IPs somehow, and know the port by convention.
#### DNS: Create a headless service with the same pod selector, and then discover DaemonSets using the endpoints resource or retrieve multiple A records from DNS.
#### Service: Create a service with the same Pod selector, and use the service to reach a daemon on a random node. (No way to reach specific node.
### Alternatives to DaemonSet
Init Scripts
It is certainly possible to run daemon processes by directly starting them on a node (e.g. using init, upstartd, or systemd). This is perfectly fine. However, there are several advantages to running such processes via a DaemonSet:

Ability to monitor and manage logs for daemons in the same way as applications.
Same config language and tools (e.g. Pod templates, kubectl) for daemons and applications.
Running daemons in containers with resource limits increases isolation between daemons from app containers. However, this can also be accomplished by running the daemons in a container but not in a Pod (e.g. start directly via Docker).
Bare Pods
It is possible to create Pods directly which specify a particular node to run on. However, a DaemonSet replaces Pods that are deleted or terminated for any reason, such as in the case of node failure or disruptive node maintenance, such as a kernel upgrade. For this reason, you should use a DaemonSet rather than creating individual Pods.

Static Pods
It is possible to create Pods by writing a file to a certain directory watched by Kubelet. These are called static pods. Unlike DaemonSet, static Pods cannot be managed with kubectl or other Kubernetes API clients. Static Pods do not depend on the apiserver, making them useful in cluster bootstrapping cases. Also, static Pods may be deprecated in the future.

Deployments
DaemonSets are similar to Deployments in that they both create Pods, and those Pods have processes which are not expected to terminate (e.g. web servers, storage servers).

Use a Deployment for stateless services, like frontends, where scaling up and down the number of replicas and rolling out updates are more important than controlling exactly which host the Pod runs on. Use a DaemonSet when it is important that a copy of a Pod always run on all or certain hosts, and when it needs to start before other Pods.

## StatefulSets
### What is StatefulSet
StatefulSet is the workload API object used to manage stateful applications.
Note: StatefulSets are stable (GA) in 1.9.
Manages the deployment and scaling of a set of Pods, and provides guarantees about the ordering and uniqueness of these 
Pods. Like a Deployment, a StatefulSet manages Pods that are based on an identical container spec. Unlike a Deployment, 
a StatefulSet maintains a sticky identity for each of their Pods. These pods are created from the same spec, but are 
not interchangeable: each has a persistent identifier that it maintains across any rescheduling.

A StatefulSet operates under the same pattern as any other Controller. You define your desired state in a StatefulSet 
object, and the StatefulSet controller makes any necessary updates to get there from the current state.

### Using StatefulSets
StatefulSets are valuable for applications that require one or more of the following.

Stable, unique network identifiers.
Stable, persistent storage.
Ordered, graceful deployment and scaling.
Ordered, automated rolling updates.
In the above, stable is synonymous with persistence across Pod (re)scheduling. If an application doesn’t require any 
stable identifiers or ordered deployment, deletion, or scaling, you should deploy your application with a controller 
that provides a set of stateless replicas. Controllers such as Deployment or ReplicaSet may be better suited to your 
stateless need

# Services and Networking

Services enable a group of pods (with same label) to be accessible as a group and be load balanced and available to 
another pod or end user. Example: back end pods to be available to front end pods front end pods to be accessible to 
end users etc. Services enable loose coupling between micro services in our application.
There are three types of services.

Create a service yaml template using --dry-run and expose your deployment using:

    kubectl expose deployment -n name-space deployment-name --type=NodePort --port=80 --name=service-name --dry-run -o yaml >myservice.yaml


## 1) NodePort: 
A port on a pod is mapped to the nodeip same port so that the pod can be accessible at nodeip:nodePort
![service nodeport](https://imgur.com/488umts.jpg)  

Node (nodeip, nodePort) <-> Service (ip Port) <-> Pod (ip, TargetPort)
nodeip: Is the ip address of any node. If you have multiple nodes, you can use any one node's ip and it will work

sample service.yaml file

    apiVersion: v1
    kind: Service
    metadata:
        name: WebServer
    spec:
        type: NodePort
        port:
            - targetport: 80
              # Optional. This the port at which the container is listening inside the pod. If not given, equal to port
              port: 80
              # Mandatory. this is the port of the service which is mapped to the targetport above
              nodeport: 30008
              # Optional. Range 30000 to 32767. Port on the node which is mapped to the service port above
              # If not given, this will be assigned a random port 30000-32767
              # This nodePort will be valid on all nodes if the pods span multiple nodes
        selector:
            app: WebApp
            type: FrontEnd
            # both these need to match the metadata of the pods.
            # if multiple pods match this, then the svc acts as frontend to all those pods. The request by deafult
            # forwarded to one of the pods chosen randomly

If multiple pods match the selector, then the service will loadbalance and send the requests to any one of the group of
matched pods randomly. These pods can be anywhere in the cluster across many pods.
Service uses the below rules to balance load:

Algorithm: Random
SessionAffinity: True

Use the following command to create the service

    kubectl create -f service.yaml

Use the following command to get the service

    kubectl get services

## 2) ClusterIP
The service creates a virtual ip inside the cluster

 apiVersion: v1
    kind: Service
    metadata:
        name: Backend
    spec:
        type: ClusterIP # default is also clusterIP so no need to specify this line
        port:
            - targetport: 8080
              port: 80  # this will be use if you don't specify nodePort
        selector:
            app: WebApp
            type: FrontEnd  # both these need to match the metadata of the pods.

This type of service is accesible to other pods either using clusterip of kubernetes at port or simply the service
name 'Backend' at port
the below command shows that the main kubernetes service 

    kubectl describe service kubernetes

is running at port 443 but targetport is 6433

    Name:              kubernetes
    Namespace:         default
    Labels:            component=apiserver
                    provider=kubernetes
    Annotations:       <none>
    Selector:          <none>
    Type:              ClusterIP
    IP:                10.96.0.1
    Port:              https  443/TCP
    TargetPort:        6443/TCP
    Endpoints:         172.17.0.48:6443
    Session Affinity:  None
    Events:            <none>
## port forwarding

    kubectl port-forward simple-webapp-dep-5f68768c85-tgqs4 4040:8080

4040 is the localhost port and 8080 is the pod port. You can access this pod using localhost:4040

    kubectl port-forward deployment/simple-webapp-dep 4040:8080

4040 is the localhost port and 8080 is the deployment's pod  port. You can access this deployment using localhost:4040

## minkube service svcname

use this command to access the service in the browser

    
## LoadBalancer
Balances load across multiple replicas of a pod. Assigns a public ip and a port

## Ingress 
Ingress, added in Kubernetes v1.1, exposes HTTP and HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by rules defined on the ingress resource.

   internet  
----|-----    
   [ Ingress ]  
   --|-----|--  
   [ Services ] 

### Ingress controllers

Kubernetes cluster does not come with a builtin ingress controller by default. We must deploy a third party ingress controller manually. Choose between one of below solutions as a ingress controller
1) GCP HTTP(S) Load balancer for GCE
2) ngnix
are supported and maintained by kubernetes project
(Others such as istio, haproxy, contour, traefik)

Create all the a separate namespace such as ingress-space for all the below objects. (kubectl create namespace ingress-space)
ngnix ingress controller is deployed like any other k8 resurce in the cluster. It consists of:

* ConfigMap (kubectl create configmap nginx-configuration --namespace ingress-space)
* ServiceAccount (kubectl create serviceaccount ingress-serviceaccount --namespace ingress-space)
* Role (apiVersion: rbac.authorization.k8s.io/v1 kind: Role)
* RoleBinding (bind the above role with above service account)
* Deployment (kubectl create -f deploy.yaml --namespace ingress-space)
* Service (kubectl expose deployment -n ingress-space nginx-ingress-controller --type=NodePort --port=80 --name=nginx-ingress-service --dry-run -o yaml >service.yaml
            kubectl create -f service.yaml --namespace ingress-space)

    roleRef:
        apiGroup: rbac.authorization.k8s.io
        kind: Role
        name: ingress-role
    subjects:
        - kind: ServiceAccount
        name: ingress-serviceaccount

We create the deployment in namespace ingress-space using below deploy.yaml:

    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
        name: nginx-ingress-controller
    spec:
        replicas: 1
        selector:
            matchLabels:
                name: nginx-ingress # match this to template.metadata.name below
        template:
            metadata:
                name: nginx-ingress
            spec:
                containers:
                    -   name: nginx-ingress-controller
                        image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
                args:
                    -   /nginx-ingress-controller
                    - --configmap=$(POD_NAMESPACE)/nginx-configuration  # POD_NAMESPACE will be ingress-space
                    # need to create a new ConfigMap with name=nginx-configuration and store all config data
                env:
                    -   name: POD_NAME
                        valueFrom:
                            fieldRef:
                                fieldPath: metadata.name
                    -   name: POD_NAMESPACE
                        valueFrom:
                            fieldRef:
                                fieldPath: metadata.namespace
                ports:
                -   name: http
                    containerPort: 80
                -   name: https
                    containerPort: 443

create a Service using below service.yaml

    apiVersion: v1
    kind: Service
    metadata:
        name: nginx-ingress-service
        namespace: ingress-space
    spec:
        type: NodePort  # or LoadBalancer
        ports:
        -   port: 80
            targetport: 80
            name: http
            protocol: TCP
        -   port: 443
            targetport: 443
            name: https
            protocol: TCP
        selector:
            matchLabels:
                name: nginx-ingress  # must match the template.metadata.name of the deployment above


create ServiceAccount and ConfigMap as well

![ingress](https://i.imgur.com/uCgjqmJ.jpg)

### Ingress resource
Ingress resource is a set of rules and configurtions.
You can specify rules such as all traffic goes to service
or all traffic to url /path1 goes to service1 and /path2 to go to service2
or all traffic to url company.com to go to service1 and videos.company.com sub domain to go to service2 etc

kubectl get all --all-namespaces
kubectl get ingress --all-namespaces
kubectl describe ingress --namespace app-space

Ingress Resources is created using yaml definition files of kind:Ingress 
Create in the app-space namespace where all the apps (deployments,replicasets, pods) and their corresponding services are running

ingress-wear.yaml

    apiVersion: extensions/v1beta
    kind: Ingress
    metadata:
        name: Ingress-wear
    spec:
        backend:  # single backend since spec contains backend
            serviceName: wear-service
            servicePort: 80

create ingress resource

    kubectl create -f ingress-wear.yaml

Few more examples of Ingress:

    apiVersion: extensions/v1beta
    kind: Ingress
    metadata:
        name: Ingress-My-Store
    spec:
        rules:  # multiple backends since spec contains rules and each rule has different host, path and backend
            - host: www.mystore.com # if only one rule , you can ignore host field. Then traffic to any host will come here
            http:
                paths:
                -   path: shoes
                    backend:
                        serviceName: shoes-service
                        servicePort: 80
                -  path: clothes
                    backend:
                        serviceName: clothes-service
                        servicePort: 80
                    
            - host: videos.mystore.com
            http:
                paths:
                -   path: shoes-vid
                    backend:
                        serviceName: shoes-vid-service
                        servicePort: 80
                -  path: clothes-vid
                    backend:
                        serviceName: clothes-vid-service
                        servicePort: 80
    ---
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
        name: ingress-wear-watch
        namespace: app-space
    annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /
        nginx.ingress.kubernetes.io/ssl-redirect: "false"
    spec:
        rules:
        - http:
            paths:
            - path: /wear
                backend:
                serviceName: wear-service
                servicePort: 8080
            - path: /watch
                backend:
                serviceName: video-service
                servicePort: 8080

                
to see the contents of the Ingress-My-Store ingress, run the describe cmd:

    kubectl describe ingress Ingress-My-Store

## Network Policies
A network policy is a specification of how groups of pods are allowed to communicate with each other and other network endpoints.  
NetworkPolicy resources use labels to select pods and define rules which specify what traffic is allowed to the selected pods.  

## Isolated and Non-isolated Pods
By default, pods are non-isolated; they accept traffic from any source.
Pods become isolated by having a NetworkPolicy that selects them. Once there is any NetworkPolicy in a namespace selecting a particular pod, that pod will reject any connections that are not allowed by any NetworkPolicy. (Other pods in the namespace that are not selected by any NetworkPolicy will continue to accept all traffic.)

The NetworkPolicy Resource
See the NetworkPolicy for a full definition of the resource.

An example NetworkPolicy might look like this:

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978

# State Persistence
## Volumes

Volume can be used to persist the pod's data on a folder inside the node (using hostPath.path: /data, hostPath.type: Directory). Even if the pod is deleted the volume will not be deleted.
hostPath type of volumes create the volumes pointing to a directory on the node so it will work good on a single node cluster.
On a multi node cluster, the pod will use /data of the node its residing on.

    apiVersion: v1
    kind: Pod
    metadata:
    name: configmap-pod
    spec:
    containers:
        - name: test
        image: busybox
        volumeMounts:
            - name: config-vol
            mountPath: /etc/config
    volumes:
        - name: config-vol
        configMap:
            name: log-config
            items:
            - key: log_level
                path: log_level
                
## Persistent Volume

    apiVersion: v1
    kind: PersistentVolume
    metadata:
        name: pv-vol1
    spec:
        accessModes: 
        - ReadWriteMany
        capacity:
            storage: 100Mi
        hostPath:
            path: /temp/data

        kubectl get persistentvolume

## Persistent Volume Claim
pvc can use matchLabels to match multiple PVs
calim one-to-one volume

    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
        name: claim-log-1
    spec:
        accessModes: 
        - ReadWriteMany
        resources:
            requests:
                storage: 100Mi

###  get persistent volumes

        kubectl get persistentvolume
## use persistent volume claim inside pod

    kind: Deployment
    spec:
      selector:
        matchLabels:
          app: mysql
      template:
        metadata:
          labels:
            app: mysql
        spec:
          volumes:
          - name: mysql-persistent-storage      <----|
            persistentVolumeClaim:                   |
              claimName: claim-log-1                 |
          containers:                                |
          - name: mysqlcontainer                     |
            image: mysql:5.7                         |
            volumeMounts:                            |
            - mountPath: "/var/lib/mysql"            |
              name: mysql-persistent-storage <-------|
              