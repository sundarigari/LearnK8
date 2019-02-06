# This exam curriculum includes these general domains and their weights on the exam:

13% – Core Concepts  
18% Configuration  
10% Multi-Container Pods  
18% – Observability  
20% – Pod Design  
13% – Services & Networking  
8% – State Persistence  


# ReplicationController vs ReplicaSet
    ReplicaSet is apiVersion apps/vi. RS needs selector/matchLabels whereas ReplicationController it's optional.  
# labels
    labels are defined for pods  
    a service or controller can use selector/matchLabels section to filter by key=value to select pods to controll.  
# create replicaset
    kubetl create -f rs-def.yaml
# get replicasets
    kubectl get rs  
    kubectl get rs rsname -o yaml > rsname.yaml

# edit replicasets
    kubectl edit replicaset rsname # will extract and open yaml in vi. Edit and save to apply.  
    # but the existing pods need to be deleted   
    # or the rs itself needs to be deleteted and recreated  
    # to have new pods with new changes

# delete replicasets
    kubectl delete replicaset rs-name  
# scale replicaset
    kubectl scale --replicas=N rsdef.yaml  
    kubectl scale rs rs-name --replicas=N
# update/replace replicaset
    kubectl replace -f rsdef.yaml  

# Deployments
    use apiVesrion apps/v1  
    use kind: Deployment  
    template has pod template  
    Deployment creates a replicaset  
 
    Can create pods, perform rolling updates and rollback if needed.  

#namespaces
    namespaces have 1) permission policies 2) quota such as max number od nodes/services/deployments etc.  
    you can access resources across namespaces using fully qualified name such as : objectname.namespace-name.svc.cluster.local
    first create Namespace object using  
    createns.yaml  
    ----  
    apiversion: v1  
    kind: Namespace  
    metadata   
        name: dev  

    or simply kubectl create namespace dev  


    two ways to specify namespace while creating an object such as pod  
    1_ kubectl create -f zzz.yaml --namespace dev  
    2_ or use metadata.namespace in the yaml to specify   
    metadata  
        name:   mypod  
        namespace: dev  

    how to set default namespace to dev?  
    --------  
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
    containers are meant to run a task to completion and exit  
    Once the task is completed the container exits  
    the cmd statement in the dockerfile specifys what command to run to completion  
    CMD "ngnix"  

    entry point  
    ENTRYPOINT "sleep"  
    with entrypoint just just pass the parameters to the entrypoint (without mentioning the entrypoint itself) in the cmd  
    docker run my-sleper-image 20  
    ENTRYPOINT is always prepended to CMD  

    FROM ubuntu  
    ENTRYPOINT ["sleep"]  
    CMD ["20"]  

    you can simple run using dockr run my-image  
    you can override the entrypoint using   
    docker run --entrypoint sleep2.0  

    how to pass command arguments from yaml in kubernetes?  
    use spec.containers.args: ["arg1", "arg2"]  
    spec  
        containers  
            -   name: ccc  
                args: ["20"]  

    how to override the entrypoint of docker from yaml in kubernetes?  
    use spec.containers.command: ["sleep"]  
    spec  
        containers  
            -   name: ccc  
                command: ["sleep2.0"]   
                args: ["20"]  
# env in container
    spec  
        containers  
        -   name  
            env:  
                -   name: key1  
                    value:  val1  
                -   name: key2  
                    value: val2  
            
# ConfigMaps
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
    Create: kubectl create -f dep.yaml  
    Get:    kubectl get deployments  
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
                completions: 3  # pods are created sequentially. 2nd pod is created after 1st pod is finished. If a pod fails then it creates a new one   until it has 3 successful creations  
                parallelism: 3  # run three parallel     
                template:  
                    spec:  
                        containers:  
                        - name: math-add  
                        image: ubuntu  
                        command: ["expr","3", "+", "2"]  
                        restartPolicy: Never  


### ┌───────────── minute (0 - 59)
### │ ┌───────────── hour (0 - 23)
### │ │ ┌───────────── day of the month (1 - 31)
### │ │ │ ┌───────────── month (1 - 12)
### │ │ │ │ ┌───────────── day of the week (0 - 6) (Sunday to Saturday;
### │ │ │ │ │                                   7 is also Sunday on some systems)
### │ │ │ │ │
### │ │ │ │ │
### * * * * * command to execute  