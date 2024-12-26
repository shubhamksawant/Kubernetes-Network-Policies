### Service discovery and DNS

The cluster must not allow for ANY Ingress or Egress as a default.

Communications are only allowed to be opened to pods where absolutely necessary. The current application consists of a website, a database, and, also for compliance reasons, a backup system.

The database must be able to accept communication from both the website and the backup system.
The website needs to be able to accept ingress on port 80.
The backup server needs to be able to egress to the NFS.
Identify your resources
The pods and the namespaces have been labeled as follows:

Website:
role: website

Database:
role: database

Backup System:
role: backup-system


The Pods have been deployed as deployments to better support scaling, updates and rollbacks as new versions of the applications are developed.

Now the company needs you to create a way to have a stable way to connect to your database and website services throughout the cluster.

--------------------------------------
```bash
apiVersion: v1
kind: Namespace
metadata:
  name: database
  labels:
    role: database

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-deployment
  namespace: database
spec:
  replicas: 1
  selector:
    matchLabels:
      role: database
  template:
    metadata:
      labels:
        role: database
    spec:
      containers:
      - name: mysql
        image: mysql:latest
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "PASSWORD123"
        - name: MYSQL_DATABASE
          value: "XDB"

---
apiVersion: v1
kind: Namespace
metadata:
  name: website
  labels:
    role: website

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: website-deployment
  namespace: website
spec:
  replicas: 1
  selector:
    matchLabels:
      role: website
  template:
    metadata:
      labels:
        role: website
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80

  ```

--------------------

<details>
<summary>
Senario 1-
Create a service for the database based on the deployment in the database namespace. Use the following details for creating service:

Name of the service: mysql-service
Selectors: role: database
Port name: mysql
Port and TargetPort: 3306
</summary>

```bash
---
apiVersion: v1 
kind: Service
metadata:
  name: mysql-service
  namespace: database
spec:
  selector: 
    role: database
  ports:
  - name: mysql
    port: 3306
    targetPort: 3306

  ```
  </details>
  
----

<details>
<summary>
Senario 2-
Create a service for the website based on the deployment in the website namespace. Use the following details for creating service:

Name of the service: website-service
Selectors: role: website
Port name: http
Port and TargetPort: 80

</summary>

```bash
---
apiVersion: v1 
kind: Service
metadata:
  name: website-service
  namespace: website
spec:
  selector: 
    role: website
  ports:
  - name: http
    port: 80
    targetPort: 80

  ```
</details>

-------------------
-------------------
In Kubernetes, when you create a Service to expose one or more pods, Kubernetes automatically creates environment variables in each pod to hold information about the service. These environment variables allow pods to dynamically discover and communicate with each other without needing to hard-code IP addresses or service names.

How Environment Variables for Services are Created
When a service is created in Kubernetes, it gets assigned a fixed IP address (the ClusterIP) and a DNS name. Kubernetes then generates environment variables based on this service for all running pods in the namespace, allowing these pods to reference the service easily. Here’s how these environment variables are typically structured:

Service Name in Uppercase: The name of the service, converted to uppercase.
Service Name Suffixes:
_SERVICE_HOST: Contains the IP address of the service.
_SERVICE_PORT: Contains the port number that the service exposes.
For example, if you have a service named website-service in your Kubernetes cluster, Kubernetes generates the following environment variables for each pod in the same namespace:

WEBSITE_SERVICE_SERVICE_HOST: The IP address of the website-service.
WEBSITE_SERVICE_SERVICE_PORT: The port number on which website-service is listening.
Example
Let's assume you have a service named database-service in the database namespace, exposing a MySQL database. You might access this service from another pod in the same namespace using environment variables like this:

mysql -h $DATABASE_SERVICE_SERVICE_HOST -P $DATABASE_SER

----------------------
----------------------

<details>
<summary>
Senario 3-
Create a new pod in the website namespace and attach an interactive shell. Retrieve the value of the WEBSITE_SERVICE_SERVICE_HOST environment variable and select the correct from the options ?
</summary>
```bash
kubectl run test --image nginx -n website
pod/test created

kubectl exec -n website test -it -- bash
root@test:/# echo $WEBSITE_SERVICE_SERVICE_HOST
10.108.140.230
root@test:/# export
declare -x DYNPKG_RELEASE="1~bookworm"
declare -x HOME="/root"
declare -x HOSTNAME="test"
declare -x KUBERNETES_PORT="tcp://10.96.0.1:443"
declare -x KUBERNETES_PORT_443_TCP="tcp://10.96.0.1:443"
declare -x KUBERNETES_PORT_443_TCP_ADDR="10.96.0.1"
declare -x KUBERNETES_PORT_443_TCP_PORT="443"
declare -x KUBERNETES_PORT_443_TCP_PROTO="tcp"
declare -x KUBERNETES_SERVICE_HOST="10.96.0.1"
declare -x KUBERNETES_SERVICE_PORT="443"
declare -x KUBERNETES_SERVICE_PORT_HTTPS="443"
declare -x NGINX_VERSION="1.27.3"
declare -x NJS_RELEASE="1~bookworm"
declare -x NJS_VERSION="0.8.7"
declare -x OLDPWD
declare -x PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
declare -x PKG_RELEASE="1~bookworm"
declare -x PWD="/"
declare -x SHLVL="1"
declare -x TERM="xterm"
declare -x WEBSITE_SERVICE_PORT="tcp://10.108.140.230:80"
declare -x WEBSITE_SERVICE_PORT_80_TCP="tcp://10.108.140.230:80"
declare -x WEBSITE_SERVICE_PORT_80_TCP_ADDR="10.108.140.230"
declare -x WEBSITE_SERVICE_PORT_80_TCP_PORT="80"
declare -x WEBSITE_SERVICE_PORT_80_TCP_PROTO="tcp"
declare -x WEBSITE_SERVICE_SERVICE_HOST="10.108.140.230"
declare -x WEBSITE_SERVICE_SERVICE_PORT="80"
declare -x WEBSITE_SERVICE_SERVICE_PORT_HTTP="80"
  ```
</details>
  
-------------------
-------------------
In Kubernetes, a DNS record is automatically created for each service, following a specific format that includes the service name, namespace, and a default domain which is typically svc.cluster.local. This DNS naming convention ensures that each service can be uniquely identified across the cluster.

For a service named database-service in the database namespace, the full DNS name would be:

database-service.database.svc.cluster.local

The above can also be used with short DNS name as follows:
database-service.database

-----------------------------
-------------------

<details>
<summary>
Senario 4-
Launch the pod in the database namespace in interactive shell and use DNS name to connect to the website-service. Which is the correct way of accessing service using DNS name.
</summary>
```bash
kubectl run test --image nginx -n database
pod/test created

kubectl exec -n database test -it -- bash
 curl website-service.website

 curl website-service.website.svc.cluster.local

  ```
</details>

  
 <details>
 <summary>
Senario 5-
Launch the pod website-deployment-{POD_HASH} in the website namespace and get the DNS nameserver and the search domains.

</summary>
```bash
k exec -it pod/website-deployment-8659cb8666-k26tg  -n website -- sh
 cat /etc/resolv.conf 
search website.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
  ```
</details>

### Service Types
--------------
company is getting closer to having an MVP ready and wants to begin testing within the company. They are looking to you to get other people within the company access to be able to test the application and also help with adding some additional routing within the cluster.

```bash

apiVersion: v1
kind: Namespace
metadata:
  name: database
  labels:
    role: database

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-deployment
  namespace: database
spec:
  replicas: 1
  selector:
    matchLabels:
      role: database
  template:
    metadata:
      labels:
        role: database
    spec:
      containers:
      - name: mysql
        image: mysql:latest
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: PASSWORD123
        - name: MYSQL_DATABASE
          value: XDB

---
apiVersion: v1
kind: Namespace
metadata:
  name: website
  labels:
    role: website

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: website-deployment
  namespace: website
spec:
  replicas: 3
  selector:
    matchLabels:
      role: website
  template:
    metadata:
      labels:
        role: website
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: database
spec:
  selector:
    role: database
  ports:
  - name: mysql
    protocol: TCP
    port: 3306
    targetPort: 3306

---
apiVersion: v1
kind: Service
metadata:
  name: website-service
  namespace: website
spec:
  selector:
    role: website
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80

  ```
-------------------

 <details>
 <summary>
Senario 1-
Re-create the current database service named mysql-service and make it a headless service.
Be sure to keep the service name the same because some applications might rely on this for DNS and service discovery.
Note: You might have to delete and re-create the service

</summary>
```bash
kubectl delete -n database svc mysql-service

---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: database
spec:
  selector:
    role: database
  ports:
  - name: mysql
    protocol: TCP
    port: 3306
    targetPort: 3306
  clusterIP: None
  ```
</details>

 <details>
 <summary>
Senario 2-
Create a nodeport service for the website with name website-service-nodeport based on the existing website-service in the website namespace. Use the following details for creating service:

Service name: website-service-nodeport
port name: http
NodePort: 30000
Selector: role: website

Test it to make sure it's reachable.

</summary>
```bash
---
apiVersion: v1
kind: Service
metadata:
 name: website-service-nodeport
 namespace: website
spec:
 ports:
 - name: http
   nodePort: 30000
   port: 80
   protocol: TCP
   targetPort: 80
 selector:
   role: website
 type: NodePort


 curl localhost:30000
  ```
</details>

 <details>
 <summary>
Senario 3-
Create an external name type service on the website namespace called google-analytics that routes to google analytics (www.google-analytics.com).
Test it to make sure it's reachable.

</summary>
```bash
---
apiVersion: v1
kind: Service
metadata:
 name: google-analytics
 namespace: website
spec:
 type: ExternalName
 externalName: www.google-analytics.com
 ```

</details>

### Troubleshooting internal networking 


After you set up your initial PoC, you decided to go on a vacation because you have been working long stressful hours.
While you were away, the Junior Admin needed to deploy new application to the cluster. But he accidentally modified some of your manifests and completely modified cluster networking to the point where the website is not working anymore.
He panicked and can’t remember everything that was modified. Unfortunately, you had not yet set up GitOps, so the Junior Admin’s changes are permanent, and you will need to troubleshoot your way through to get the network back to ensure the website works properly again.
You will begin troubleshooting the network by investigating the CNIs for errors.

```bash
apiVersion: v1
kind: Namespace
metadata:
  name: website
  labels:
    app: website

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: website-deployment
  namespace: website
spec:
  replicas: 3
  selector:
    matchLabels:
      app: website
  template:
    metadata:
      labels:
        app: website
    spec:
      containers:
      - name: website
        image: wbassler/flask-app-example:v0.0.1
        imagePullPolicy: Always
        ports:
        - containerPort: 5000
        env:
        - name: MYSQL_HOST
          value: "mysql.database.svc.cluster.local"
        - name: MYSQL_USER
          value: "root"
        - name: MYSQL_PASSWORD
          value: "PASSWORD123"
        - name: MYSQL_DB
          value: "userdb"

---
apiVersion: v1
kind: Service
metadata:
  name: website-service-nodeport
  namespace: website
spec:
  type: NodePort
  ports:
  - port: 5000
    targetPort: 5000
    nodePort: 30000
    name: http
  selector:
    role: website

---
apiVersion: v1
kind: Namespace
metadata:
  name: database
  labels:
    role: database

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-init-db
  namespace: database
data:
  init-db.sql: |
    CREATE TABLE IF NOT EXISTS users (
        id INT AUTO_INCREMENT PRIMARY KEY,
        username VARCHAR(255) NOT NULL,
        password VARCHAR(255) NOT NULL
    );
    INSERT INTO users (username, password) VALUES ('exampleUser', 'examplePass');

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-deployment
  namespace: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "PASSWORD123"
        - name: MYSQL_DATABASE
          value: "userdb"
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
        - name: init-db-script
          mountPath: /docker-entrypoint-initdb.d
      volumes:
      - name: mysql-storage
        emptyDir: {}
      - name: init-db-script
        configMap:
          name: mysql-init-db

---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: database
spec:
  clusterIP: None
  ports:
  - port: 3306
    targetPort: 3306
    name: mysql
    protocol: TCP
  selector:
    role: database

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: website
spec:
  podSelector: {}
  policyTypes:
  - Egress

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-website-ingress-to-database
  namespace: database
spec:
  podSelector:
    matchLabels:
      role: database
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          role: website
    - podSelector:
        matchLabels:
          role: website
    ports:
    - protocol: TCP
      port: 3306
  policyTypes:
  - Ingress

 ```

 <details>
 <summary>
Senario -
It appears that the website pods in the website namespace cannot connect to the database pod in the database namespace. Looks like it just hangs when trying to connect.
Open another terminal and execute a port forward to one of the website pods on ports 8000:5000 and execute the test-network.sh script.
Investigate the network policies in the website namespace and delete a network policy that is no longer needed.


```bash
## test-network.sh
#!/bin/bash
curl -m 3 -X POST http://localhost:8000/auth -d "username=exampleUser&password=examplePass"
 ```
 
</summary>
```bash

root@controlplane ~ ➜  sh test-network.sh 
curl: (7) Failed to connect to localhost port 8000 after 0 ms: Connection refused

root@controlplane ~ ➜ k port-forward pod/website-deployment-6d98cbcc89-bmszs 8000 5000 -n website
Forwarding from 127.0.0.1:8000 -> 8000
Forwarding from [::1]:8000 -> 8000
Forwarding from 127.0.0.1:5000 -> 5000
Forwarding from [::1]:5000 -> 5000

root@controlplane ~ ➜ k get netpol -n website
NAME                  POD-SELECTOR   AGE
default-deny-egress   <none>         15m

root@controlplane ~ ➜  k describe  netpol -n website
Name:         default-deny-egress
Namespace:    website
Created on:   2024-12-26 11:26:11 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Not affecting ingress traffic
  Allowing egress traffic:
    <none> (Selected pods are isolated for egress connectivity)
  Policy Types: Egress

root@controlplane ~ ➜ k get netpol -n database
NAME                                POD-SELECTOR    AGE
allow-website-ingress-to-database   role=database   15m

root@controlplane ~ ➜  k describe  netpol -n database
Name:         allow-website-ingress-to-database
Namespace:    database
Created on:   2024-12-26 11:26:11 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     role=database
  Allowing ingress traffic:
    To Port: 3306/TCP
    From:
      NamespaceSelector: role=website
    From:
      PodSelector: role=website
  Not affecting egress traffic
  Policy Types: Ingress

k delete netpol default-deny-egress  -n website 
networkpolicy.networking.k8s.io "default-deny-egress" deleted

 ```

</details>




 <details>
 <summary>
Senario -
It looks like the connection is still off between the website and the database.
Have a look at the database namespace and fully investigate the network policies. Edit the network policy that is not allowing ingress from the website.

</summary>
```bash
root@controlplane ~ ➜  k get netpol -n database
NAME                                POD-SELECTOR    AGE
allow-website-ingress-to-database   role=database   25m

root@controlplane ~ ➜  k describe  netpol -n database
Name:         allow-website-ingress-to-database
Namespace:    database
Created on:   2024-12-26 11:26:11 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     role=database
  Allowing ingress traffic:
    To Port: 3306/TCP
    From:
      NamespaceSelector: role=website
    From:
      PodSelector: role=website
  Not affecting egress traffic
  Policy Types: Ingress

root@controlplane ~ ➜  k get netpol -n database -o yaml
apiVersion: v1
items:
- apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"networking.k8s.io/v1","kind":"NetworkPolicy","metadata":{"annotations":{},"name":"allow-website-ingress-to-database","namespace":"database"},"spec":{"ingress":[{"from":[{"namespaceSelector":{"matchLabels":{"role":"website"}}},{"podSelector":{"matchLabels":{"role":"website"}}}],"ports":[{"port":3306,"protocol":"TCP"}]}],"podSelector":{"matchLabels":{"role":"database"}},"policyTypes":["Ingress"]}}
    creationTimestamp: "2024-12-26T11:26:11Z"
    generation: 1
    name: allow-website-ingress-to-database
    namespace: database
    resourceVersion: "24708"
    uid: e6f0378f-796c-40be-9279-3704350315a5
  spec:
    ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            role: website
      - podSelector:
          matchLabels:
            role: website
      ports:
      - port: 3306
        protocol: TCP
    podSelector:
      matchLabels:
        role: database
    policyTypes:
    - Ingress
kind: List
metadata:
  resourceVersion: ""


root@controlplane ~ ➜  k get ns --show-labels | grep website
website           Active   31m     app=website,kubernetes.io/metadata.name=website

root@controlplane ~ ➜  k get pods -n website --show-labels 
NAME                                  READY   STATUS    RESTARTS   AGE   LABELS
website-deployment-6d98cbcc89-bmszs   1/1     Running   0          32m   app=website,pod-template-hash=6d98cbcc89
website-deployment-6d98cbcc89-k4zb2   1/1     Running   0          32m   app=website,pod-template-hash=6d98cbcc89
website-deployment-6d98cbcc89-m5prr   1/1     Running   0          32m   app=website,pod-template-hash=6d98cbcc89

root@controlplane ~ ➜  k get pods -n database --show-labels 
NAME                                   READY   STATUS    RESTARTS   AGE   LABELS
database-deployment-866789c4b4-5kc2t   1/1     Running   0          32m   app=database,pod-template-hash=866789c4b4


root@controlplane ~ ➜ k edit netpol -n database
networkpolicy.networking.k8s.io/allow-website-ingress-to-database edited

root@controlplane ~ ➜ k get netpol -n database -o yaml
apiVersion: v1
items:
- apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"networking.k8s.io/v1","kind":"NetworkPolicy","metadata":{"annotations":{},"name":"allow-website-ingress-to-database","namespace":"database"},"spec":{"ingress":[{"from":[{"namespaceSelector":{"matchLabels":{"role":"website"}}},{"podSelector":{"matchLabels":{"role":"website"}}}],"ports":[{"port":3306,"protocol":"TCP"}]}],"podSelector":{"matchLabels":{"role":"database"}},"policyTypes":["Ingress"]}}
    creationTimestamp: "2024-12-26T11:26:11Z"
    generation: 3
    name: allow-website-ingress-to-database
    namespace: database
    resourceVersion: "28334"
    uid: e6f0378f-796c-40be-9279-3704350315a5
  spec:
    ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            app: website
      - podSelector:
          matchLabels:
            app: website
      ports:
      - port: 3306
        protocol: TCP
    podSelector:
      matchLabels:
        app: database
    policyTypes:
    - Ingress
kind: List
metadata:
  resourceVersion: ""

  ```
</details>


 <details>
 <summary>
Senario -
There still seems to be a connection timeout between the website and the database. But at this point, we have validated that the network policies are once again configured properly.
Since we have validated that egress from the website is working in previous steps using the test-network.sh script, let's assume that there a problem with the database.
Remember that the database is also using a service. Investigate the service mysql-service in the database namespace and make the necessary changes.
Note: Test the connection using bash test-network.sh

```bash
 sh test-network.sh 
curl: (7) Failed to connect to localhost port 8000 after 0 ms: Connection refused

root@controlplane ~ ✖ kubectl -n website run nginx -l "app=website" --image nginx
pod/nginx created

root@controlplane ~ ➜  kubectl exec -it -n website nginx -- curl mysql-service.database

curl: (6) Could not resolve host: mysql-service.database
command terminated with exit code 6

  ```

</summary>
```bash
k describe svc -n database
Name:              mysql-service
Namespace:         database
Labels:            <none>
Annotations:       <none>
Selector:          role=database
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                None
IPs:               None
Port:              mysql  3306/TCP
TargetPort:        3306/TCP
Endpoints:         <none>
Session Affinity:  None
Events:            <none>

root@controlplane ~ ➜  k edit  svc -n database
service/mysql-service edited

root@controlplane ~ ➜  k describe svc -n database
Name:              mysql-service
Namespace:         database
Labels:            <none>
Annotations:       <none>
Selector:          app=database
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                None
IPs:               None
Port:              mysql  3306/TCP
TargetPort:        3306/TCP
Endpoints:         10.0.0.23:3306
Session Affinity:  None
Events:            <none>

  ```
</details>


<details>
<summary>
Senario -
The database is now reachable internally, but we are still getting a timeout error. The only difference is we are now getting an error when trying to connect Unknown MySQL server host 'mysql.database.svc.cluster.local'.
This hostname is actually configured as an environment variable in your website-deployment manifest as MYSQL_HOST, but it cannot be resolved. This is, in fact, incorrect. Modify this environment variable to the correct value in the website deployment.

</summary>

```bash

root@controlplane ~ ➜  k describe  deployment.apps/website-deployment -n website
Name:                   website-deployment
Namespace:              website
CreationTimestamp:      Thu, 26 Dec 2024 11:26:11 +0000
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=website
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=website
  Containers:
   website:
    Image:      wbassler/flask-app-example:v0.0.1
    Port:       5000/TCP
    Host Port:  0/TCP
    Environment:
      MYSQL_HOST:      mysql.database.svc.cluster.local
      MYSQL_USER:      root
      MYSQL_PASSWORD:  PASSWORD123
      MYSQL_DB:        userdb
    Mounts:            <none>
  Volumes:             <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   website-deployment-6d98cbcc89 (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  50m   deployment-controller  Scaled up replica set website-deployment-6d98cbcc89 to 3

root@controlplane ~ ➜  k edit deployment.apps/website-deployment -n website
deployment.apps/website-deployment edited

root@controlplane ~ ➜  k describe  deployment.apps/website-deployment -n website
Name:                   website-deployment
Namespace:              website
CreationTimestamp:      Thu, 26 Dec 2024 11:26:11 +0000
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               app=website
Replicas:               3 desired | 2 updated | 4 total | 3 available | 1 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=website
  Containers:
   website:
    Image:      wbassler/flask-app-example:v0.0.1
    Port:       5000/TCP
    Host Port:  0/TCP
    Environment:
      MYSQL_HOST:      mysql-service.database.svc.cluster.local
      MYSQL_USER:      root
      MYSQL_PASSWORD:  PASSWORD123
      MYSQL_DB:        userdb
    Mounts:            <none>
  Volumes:             <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    ReplicaSetUpdated
OldReplicaSets:  website-deployment-6d98cbcc89 (2/2 replicas created)
NewReplicaSet:   website-deployment-5d94b7bd99 (2/2 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  52m   deployment-controller  Scaled up replica set website-deployment-6d98cbcc89 to 3
  Normal  ScalingReplicaSet  5s    deployment-controller  Scaled up replica set website-deployment-5d94b7bd99 to 1
  Normal  ScalingReplicaSet  0s    deployment-controller  Scaled down replica set website-deployment-6d98cbcc89 to 2 from 3
  Normal  ScalingReplicaSet  0s    deployment-controller  Scaled up replica set website-deployment-5d94b7bd99 to 2 from 1


  ```
</details>



<details>
<summary>
Senario -
As the internal networking is now resolved and the website can connect to the database, we need to resolve the connection issue for external networking.
Users are complaining that the URL they used to use to test the site is not responding.
We originally configured this to be set up with a nodePort service type. Knowing this, we need to investigate the service used for the website. Make necessary changes to resolve the website nodeport type service website-service-nodeport.

</summary>

```bash

root@controlplane ~ ➜  k describe service/website-service-nodeport -n website
Name:                     website-service-nodeport
Namespace:                website
Labels:                   <none>
Annotations:              <none>
Selector:                 role=website
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.101.52.136
IPs:                      10.101.52.136
Port:                     http  5000/TCP
TargetPort:               5000/TCP
NodePort:                 http  30000/TCP
Endpoints:                <none>
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>

root@controlplane ~ ➜  k edit service/website-service-nodeport -n website
service/website-service-nodeport edited

root@controlplane ~ ➜  k describe service/website-service-nodeport -n website
Name:                     website-service-nodeport
Namespace:                website
Labels:                   <none>
Annotations:              <none>
Selector:                 app=website
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.101.52.136
IPs:                      10.101.52.136
Port:                     http  5000/TCP
TargetPort:               5000/TCP
NodePort:                 http  30000/TCP
Endpoints:                10.0.0.102:5000,10.0.0.190:5000,10.0.1.126:5000 + 1 more...
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>


curl http://localhost:30000
                    
        <!DOCTYPE html>
        <html lang="en">
        <head>
            <meta charset="UTF-8">
            <meta name="viewport" content="width=device-width, initial-scale=1.0">
            <title>Login - Company X</title>
            <style>
                body, html {
                    height: 100%;
                    margin: 0;

  ```
</details>



