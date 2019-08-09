

## Local storage for elasticsearch

Local에서 elasticsearch 구성을 위해 volume 설정은 다음과 같다.

1. Storage Class 생성
2. Persistent Volume 생성
3. StatefulSet으로 elasticsearch 구성



### Storage Class 생성

https://kubernetes.io/docs/concepts/storage/storage-classes/#local

Volume Plugin을 Local로 생성한다.

( storage class name : es-storage )

```bash
$ vi sc_es5.yaml

kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: es-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer


$ kubectl create -f sc_es5.yaml

$ kubectl get sc
NAME         PROVISIONER                    AGE
es-storage   kubernetes.io/no-provisioner   7s
```



### Persistent Volume 생성

worker node가 3대일 경우 각 노드마다 /mnt/disks/vol2  디렉토리 미리 생성

```
# worker node 3대 모두 디렉토리 생성
$ mkdir -p /mnt/disks/vol2
```



https://kubernetes.io/docs/concepts/storage/volumes/#local

위에서 생성한 storage class(es-storage)를 storageClassName 항목에 세팅.

local volume은 nodeAffinity 항목이 필수이다.

(local PersistentVolume은 사용자가 필요 없을 시 직접 삭제 필요.)

```bash
$ vi pv_es.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: es-pv01
spec:
  capacity:
    storage: 500Mi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: es-storage
  local:
    path: /mnt/disks/vol2
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: es-pv02
spec:
  capacity:
    storage: 500Mi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: es-storage
  local:
    path: /mnt/disks/vol2
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node2
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: es-pv03
spec:
  capacity:
    storage: 500Mi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: es-storage
  local:
    path: /mnt/disks/vol2
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node3
          
$ kubectl create -f pv_es.yaml
persistentvolume/es-pv01 created
persistentvolume/es-pv02 created
persistentvolume/es-pv03 created

$ kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
es-pv01   500Mi      RWO            Retain           Available           es-storage              30s
es-pv02   500Mi      RWO            Retain           Available           es-storage              30s
es-pv03   500Mi      RWO            Retain           Available           es-storage              30s
```



### StatefulSet으로 elasticsearch 구성

elasticsearch 5.6.4 버전이며 replicas: 3 으로 구성

volumeClaimTemplates 영역에서 위에서 생성한 es-storage 세팅

```bash
$  vi 01-es_sts.yaml

apiVersion: apps/v1beta1         # API version of kubernetes in which `StatefulSet` is available. For Kubernetes 1.8.7 its apps/v1beta1
kind: StatefulSet                # Type of resource that we are creating
metadata:                        # Holds metadata for this resource
  name: es                       # Name of this resource
  labels:                        # Extra metadata goes inside labels. It is for stateful resource
    component: elasticsearch     # Just a metadata we are adding
spec:                            # Holds specification of this resource
  replicas: 3                    # Responsible for maintaining the given number of replicas
  serviceName: elasticsearch     # Name of service, required by statefulset
  template:                      # Template holds the spec of the pod that will be created and maintained by statefulset
    metadata:                    # Holds metadata for the pod
      labels:                    # Extra metadata goes inside labels. It is for the pod
        component: elasticsearch # Just a metadata for the pod
    spec:                        # Holds the spec of the pod
      initContainers:            # will always initialize before other containers in the pod
      - name: init-sysctl        # Name of the init-container
        image: busybox           # Image that will be deployed in this container
        imagePullPolicy: IfNotPresent              # Sets the policy that only pull image from registry if it is not available locally
        command: ["sysctl", "-w", "vm.max_map_count=262144"]  # Sets the system varibale in the container, this value is required by ES
        securityContext:         # Security context holds any special permission given to this container
          privileged: true       # This container gets the right to run in privilaged mode
      containers:                # Holds the list and configs of normal containers in the pod
      - name: es                 # Name of the first container
        securityContext:         # Security context holds any special permission given to this container
          capabilities:          # Container will have the capability to IPC Lock , can lock on memory so that it is not swapped out.
            add:
              - IPC_LOCK
        image: quay.io/pires/docker-elasticsearch-kubernetes:5.6.4 # specifies the image of elasticsearch to be installed
        env:                     # array of environment variables with values are passed to this image
        - name: KUBERNETES_CA_CERTIFICATE_FILE
          value: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: "CLUSTER_NAME"
          value: "nexledger-cluster"
        - name: "DISCOVERY_SERVICE"
          value: "elasticsearch"
        - name: NETWORK_HOST
          value: "0.0.0.0"
        - name: ES_JAVA_OPTS           #Specify the Heap Size
          value: -Xms512m -Xmx512m
        ports:                         # Ports that this pod will open
        - containerPort: 9200
          name: http
          protocol: TCP
        - containerPort: 9300
          name: transport
          protocol: TCP
        volumeMounts:                 # The path where volume will be mounted.
        - mountPath: /data
          name: storage               # Name given to this mount
  updateStrategy:
    type: RollingUpdate
  volumeClaimTemplates:               # It  provides stable storage using PersistentVolumes provisioned by a PersistentVolume Provisioner
  - metadata:                         # Metadata given to this resource (Persistant Volume Claim)
      name: storage                   # Name of this resource
    spec:                             # Specification of this PVC (Persistant Volume Claim)
      storageClassName: "es-storage"  # Storage class used to provision this PVC
      accessModes: [ ReadWriteOnce ]  # Access mode of the volume
      resources:                      # Holds the list of resources
        requests:                     # Requests sent to the storage class
          storage: 500Mi                # Request to provision a volume of 1 GB
---
apiVersion: v1               #API Version of the resource
kind: Service                #Type of resource
metadata:                    #Contains metadata of this resource.
  name: elasticsearch        #Name of this resource
  labels:                    #Additional identifier to put on pods
    component: elasticsearch #puts component = elasticsearch
spec:                        #Specifications of this resource
  type: ClusterIP            #type of service
  selector:                  #will distribute load on pods which
    component: elasticsearch #have label `component = elasticsearch`
  ports:                     #Port on which LoadBalancer will listen
  - name: http               #Name given to port
    port: 9200               #Port number
    protocol: TCP            #Protocol supported
  - name: transport          #Name given to port
    port: 9300               #Port number
    protocol: TCP            #Protocol supported


$ kubectl create -f 01-es_sts.yaml
statefulset.apps/es created
service/elasticsearch created

$ k get all
NAME       READY   STATUS    RESTARTS   AGE
pod/es-0   1/1     Running   0          19s
pod/es-1   1/1     Running   0          16s
pod/es-2   1/1     Running   0          12s

NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/elasticsearch   ClusterIP   172.18.125.245   <none>        9200/TCP,9300/TCP   19s
service/kubernetes      ClusterIP   172.18.0.1       <none>        443/TCP             15h

NAME                  READY   AGE
statefulset.apps/es   3/3     19s

```



elasticsearch 상태 확인

```bash
# elasticsearch port 확인 : 9200
$ k get svc
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
elasticsearch   ClusterIP   172.18.125.245   <none>        9200/TCP,9300/TCP   72s
kubernetes      ClusterIP   172.18.0.1       <none>        443/TCP             15h

# 임의의 pod에 접속하기 위해 port forwarding
$ k get pod
NAME   READY   STATUS    RESTARTS   AGE
es-0   1/1     Running   0          97s
es-1   1/1     Running   0          94s
es-2   1/1     Running   0          90s
$ k port-forward elasticsearch 9200:9200 &

# health check
$ curl -XGET 'localhost:9200/_cat/health?v&pretty'
Handling connection for 9200
epoch      timestamp cluster           status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1564705247 00:20:47  xx-cluster green           1         1      0   0    0    0        0             0                  -                100.0%
```

