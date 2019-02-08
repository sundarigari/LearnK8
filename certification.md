# This exam curriculum includes these general domains and their weights on the exam:

| Weight| Topic                   |
|-------|-------------------------|
| 13%   | Core Concepts           |
| 18%   | Configuration           |
| 10%   | Multi-Container Pods    |
| 18%   |  Observability          |
| 20%   |  Pod Design             |
| 13%   |  Services & Networking  |
| 8%    |  State Persistence      |


# ReplicationController and ReplicaSet
A ReplicaSet ensures that a specified number of pod replicas are running at any given time. However, a Deployment is a higher-level concept that manages ReplicaSets and provides declarative updates to Pods along with a lot of other useful features. Therefore, we recommend using Deployments instead of directly using ReplicaSets, unless you require custom update orchestration or don’t require updates at all.

This actually means that you may never need to manipulate ReplicaSet objects: use a Deployment instead, and define your application in the spec section.

Example
frontend.yaml 

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
            
Note: A Deployment that configures a ReplicaSet is now the recommended way to set up replication as opposed to using ReplicationController. A ReplicationController ensures that a specified number of pod replicas are running at any one time. In other words, a ReplicationController makes sure that a pod or a homogeneous set of pods is always up and available. ReplicaSet is apiVersion apps/v1. RS needs selector/matchLabels whereas for ReplicationController it's optional.  

## labels
labels are defined for k8 objects such as pods. A service or controller can use selector/matchLabels section to filter by key=value to select pods to control.  

## create replicaset
    kubectl create -f rs-def.yaml

## get replicasets
    kubectl get rs  
    kubectl get rs rsname -o yaml > rsname.yaml

## edit replicasets
    kubectl edit replicaset rsname 
    
will extract and open yaml in vi. Edit and save to apply.  Make sure the existing pods need to be deleted   
or the rs itself needs to be deleteted and recreated to have new pods with new changes

## delete replicasets
    kubectl delete replicaset rs-name  

## scale replicaset
    kubectl scale --replicas=N rsdef.yaml  
    kubectl scale rs rs-name --replicas=N

## update/replace replicaset
    kubectl replace -f rsdef.yaml  

# Deployments

    apiVesrion: apps/v1  
    kind: Deployment  
    template: # has pod spec  
        spec:
            containers:
            - name:

Deployment creates a replicaset.
Can create pods, perform rolling updates and rollback if needed.  

# Namespaces
namespaces have 1) permission policies 2) quota such as max number od nodes/services/deployments etc.  
You can access resources across namespaces using fully qualified name such as : objectname.namespace-name.svc.cluster.local  
first create Namespace object using  

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

# Docker commands

containers are meant to run a task to completion and exit.  
Once the task is completed the container exits  
the cmd statement in the dockerfile specifys what command to run to completion  

    CMD "ngnix"  

entry point   

    ENTRYPOINT "sleep"  
    
with entrypoint just pass the parameters to the entrypoint (without mentioning the entrypoint itself) in the cmd  
   
    docker run my-sleper-image 20  
    
ENTRYPOINT is always prepended to CMD  

    FROM ubuntu  
    ENTRYPOINT ["sleep"]  
    CMD ["20"]  

you can simply run using 

    dockr run my-image  

you can override the entrypoint using   

    docker run --entrypoint sleep2.0  

how to pass command arguments from yaml in kubernetes?  use spec.containers.args: ["arg1", "arg2"]  

    spec  
        containers  
            -   name: ccc  
                args: ["20"]  

how to override the entrypoint of docker from yaml in kubernetes?  use spec.containers.command: ["sleep"]  

    spec  
        containers  
            -   name: ccc  
                command: ["sleep2.0"]   
                args: ["20"]  

k8 command is smae as docker ENTRYPOINT
k8 args is same as docker CMD

## env in container
    spec  
        containers  
        -   name  
            env:  
                -   name: key1  
                    value:  val1  
                -   name: key2  
                    value: val2  
            
# ConfigMaps
ConfigMaps allow you to decouple configuration artifacts from image content to keep containerized applications portable.

    kubectl create configmap mycmp --form-literal=key1=val1 --from-literal=key2=val2  
    kubectl create -f mycmp.yaml  

    apiVersion: v1  
    kind: Configmap  
    metadata:  
        name: mycmp  
    data:  
        key1:val1  
        key2:val2  

    kubectl get configmaps  
    kubectl describe configmaps  

## inject configmap data into pod
    use spec.containers.envFrom.configmapRef.name=nameofthe-config-map  
    use spec.containers.envFrom.configmapRef.key=nameofthe-key  


### to inject the entire configmap into a container spec
    spec  
        containers  
        -   name  
            envFrom:  
                -   configMapRef:   
                        name: configmap1  
                -   configMapRef:   
                        name: configmap2  

### to inject a single key from a configmap into a container spec
    spec  
        containers  
        -   name  
            env:  
            -   name: APP_COLOR  
                valueFrom:    
                    configMapKeyRef:  
                        name: app-config  
                        key:  aAPP_COLOR  

# Secrets
    
    kubectl create secret generic mypassword --from-literal=REDIS_PASS=!@#$%^&GGHJ   
    kubectl create secret generic mypassword --from-file=mysecrets.properties    

yaml

    apiVersion: v1  
    kind: Secret  
    metadata:  
        name: app-secrets  
    data:  
        REDIS_PASS: base64-encoded-pass  
        REDIS_HOST: base64-encoded-hostname  

    echo -n mypasword | base64  
    bXlwYXN3b3Jk  
    echo -n bXlwYXN3b3Jk | base64  --decode  
    mypassword  

### to inject the entire secrets into a container spec
    spec  
        containers  
        -   name  
            envFrom:  
                -   secretRef:   
                        name: mysecrets1  
                -   secretRef:   
                        name: mysecrets2   


### to inject a single key from a secrets into a container spec
    spec
        containers
        -   name
            env:
            -   name: REDIS_PASS
                valueFrom:  
                    secretKeyRef:
                        name: app-secrets
                        key:  REDIS_PASS

#  Security

    Pod security context specifies the context for security od the pods. 
    Container scurity context specifies the context for the container. Container context always overrides Pod security context.

## security context at pod level
    spec
        containers
        -   name: c1
        -   name: c2
        securityContext:
            runAsUer: 1000
            capabilities:
                add: ["MAC_ADMIN"]

## security context for the container
    spec
        containers
        -   name: c1
            securityContext:
                runAsUer: 1000
                capabilities:
                    add: ["MAC_ADMIN"]
        -   name: c2
            securityContext:
                runAsUer: 1001
                capabilities:
                    add: ["PC_ADMIN"]

# service accounts
RBAC can be used to a assign roles to a sa
authentication
todo

Kubernetes has 1)user account Ex: admin, developer etc  used by humans
2)service account ex: Prometheus Jenkins used by application to interact with kubernetes api 

kubectl create serviceccount dashboard-sa
kubectl get serviceaccount

serviceaccount creates automatically a token, creates a secret object, copies the token into the secret object and then links the secret to the sa.
To see the token issue kubectl describe secret command for the secret used by the sa

even if the third party application is not running inside the cluster, it can call the k8 api using this token.
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
                serviceAccountName: saname
                containers:
                -
when you change the sa for a pod, you have to delete the pod and recreate it to take effect.
Incase of a deployment, no need to delete the pod. Just re apply the dep. yaml

you can choose to not mount any sa into a pod by using

    spec
        automountServiceAccountToken: false

# Resource Requirements
Min amount of requirement for a pod to run: .5 cpu and 256Mi mem
Ki = 1024 K=1000
Mi=1048576 M=1000,000
default limit for pod: 1cpu 512Mi

you can change the resources and limits in the spec  

    spec  
        resources:  
            requests:  
                memory:  
                cpu:  
            limits:  
                memory:  
                cpu:  

# taints and tolerations
a node can be tainted (using Node 1 is tainted with: taint=blue) so that not all pods can be hosted on that node.  
Then pods can be specified to tolerate a taint. For example pod A can ve specified as tolerate=blue and pod b can be specified to tolerate=red
in such scenario only pod A will be placed in node 1.  
pod B will be placed on other nodes.  

    kubectl taint nodes node-nme key=value:NoSchedule  ## NoSchedule PreferNoSchedule NoExecute  
    kubectl taint nodes node-nme app=blue:NoSchedule  ## NoSchedule PreferNoSchedule NoExecute  

in the pod spec use tolerations  

    spec:  
        tolerations:   
        - key: "app"  
        operator: "Equal"  
        value: "blue"  
        effect: "NoSchedule"  

    kubesctl describe nore kubemaster | grep Taint  
will show that master is tainted hence no pods are scheduled to master node  

#nodeSelector
enables to choose a node (as opposed to taints which disables a node for pods )  
nodes have labels in metadata as follows:  

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
    spec:  
        nodeSelector:  
            size: Large  


labels are simple mechanism to choose a pod  
using nodeAffinity you can use complex logical operators such as In, NotIn Exists etc. to filter based on multiple lables/values  


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

# multi container pods

patterns:

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

# Readiness Probes
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
To solve this you need to add readinessProbes in the pod definition files.  

    spec:  
        readinessProbe:  
            httpGet:  
                path: /api/ready  
                port: 8080  

    spec:  
        readinessProbe:  
            tcpSocket:  
                port:  

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

# container logging
    docker logs -f dockername  # shows the logs for the container
    kubectl logs -f podname dockername  # shows the logs for the container dockername in the pod: podname

# Monitor resource consumption
CPU, memory  
Open Source: Prometeus, MetricsServer (HeapSter), ElasticStack   
proprietary: DataDog, DynaTree   

## MetricsServer
Gathers metrics from nodes, pods, summarizes them to a in-memory database    
Each node run a kubelet, which is responsible for receiving instructions from kubernetes master node and implement the instructions vis  launching/shutting down pods and containers   
cAdvisor also runs on all nodes. responsible for aggregating the metrics of the node and sending to MetS  

    minikube addons enable metrics-server  

to see the metrics. Only one metrics server per k8 cluster   

    kubectl top node  
    kubectl top pod  

# Labels and Selectors
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

# Rollout and Versioning in Deployments
Revision1 is created when a new deployment is created  
when the new image/spec applied then new Revision2 is pplied  

    kubectl rollout status deployment/depname   

## shows the status deployment depname successfully rolled out
    kubectl rollout history deployment/depname   

## shows the history of deployment depname
    deployment.extensions/redis  
    REVISION  CHANGE-CAUSE  
    1         <none>  
    2         <none>  

chaning the number of replicas does not count as new revision since we are not changing the template.  
changing image counts as new revision since we are chaning the template  

## deployment strategies
Recreate strategy: Delete all pods and re-create them with newer image. Cons: App will be unavailable for some time  
Rolling Update Strtegy: Deletes one pod and creates one new pod, then deletes another old pod and creates a new pod and so on  

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

### Rollback  
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

# Jobs  


    spec:  
        restartPolicy: Always  

The above spec will try to run the pod if it exits thus making sure the pod erver is always running  
Jobs do not run for ever. They perform a task such as adding two numbers and exit with the result copied to some text file on a log serevr  

Job is like replicaset, a set of pods except the pods run to completion and do not re-start  
job definition  

    apiVersion: batch/v1  
    kind: Job  
    metadata:  
        name: math-add-job  
    spec:  
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

# cronjobs

cronjob definition (notice jobTemplate and total 3 specs)  

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

# Services
Services enable a group of pods (with same label) to be accessible as a group and be load balanced and available to another pod or end user. Example: back end pods to be available to front end pods front end pods to be accessible to end users etc. Services enable loose coupling between micro services in our application.
There are three types of services.

Create a service yaml template using --dry-run and expose your deployment using:

    kubectl expose deployment -n name-space deployment-name --type=NodePort --port=80 --name=service-name --dry-run -o yaml >myservice.yaml


## NodePort: 
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
                port: 80  # this will be use if you don't specify nodePort
                nodeport: 30008  # range 30000 to 32767
        selector:
            app: WebApp
            type: FrintEnd  # both these need to match the metadata of the pods.

If multiple pods match the selector, then the service will loadbalance and send the requests to any one of the group of matched pods randomly. These pods can be anywhere in the cluster across many pods.
Service uses the below rules to balance load:

Algorithm: Random
SessionAffinity: True

Use the following command to create the service

    kubectl create -f service.yaml

Use the following command to get the service

    kubectl get services

## ClusterIP
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
            type: FrintEnd  # both these need to match the metadata of the pods.
This type of service is accesible to other pods either using clusterip of kubernetes at port or simply the service name 'Backend' at port

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

## LoadBalancer
Balances load across multiple replicas of a pod. Assigns a public ip and a port

# Ingress 
Ingress, added in Kubernetes v1.1, exposes HTTP and HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by rules defined on the ingress resource.

   internet  
----|-----    
   [ Ingress ]  
   --|-----|--  
   [ Services ] 

## Ingress controllers

Kubernetes cluster does not come with a builtin ingress controller by default. We must deploy a third party ingress controller manually. Choose between one of below solutions as a ingress controller
1) GCP HTTP(S) Load balancer for GCE
2) ngnix
are supported and maintained by kubernetes project
(Others such as istio, haproxy, contour, traegik)

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

## Ingress resource
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
    ----
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