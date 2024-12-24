# Pod-Pod Communication & Network Policies

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
--------------------------------------
```bash
apiVersion: v1
kind: Namespace
metadata:
  name: website
  labels:
    role: website
---
apiVersion: v1
kind: Pod
metadata:
  name: website
  namespace: website
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
kind: Namespace
metadata:
  name: database
  labels:
    role: database
---
apiVersion: v1
kind: Pod
metadata:
  name: database
  namespace: database
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
  name: backup-system
  labels:
    role: backup-system
---
apiVersion: v1
kind: Pod
metadata:
  name: backup-system
  namespace: backup-system
  labels:
    role: backup
spec:
  containers:
  - name: centos
    image: centos:latest
    command: ["sleep", "86400"]
  ```
  
root@controlplane ~/nwk ➜ k get pod/backup-system -n backup-system --show-labels
NAME            READY   STATUS    RESTARTS   AGE     LABELS
backup-system   1/1     Running   0          7m57s   role=backup

root@controlplane ~/nwk ➜ k get pod/database -n database --show-labels
NAME       READY   STATUS    RESTARTS   AGE     LABELS
database   1/1     Running   0          8m33s   role=database

root@controlplane ~/nwk ➜  k get pod/website -n website --show-labels
NAME      READY   STATUS    RESTARTS   AGE     LABELS
website   1/1     Running   0          8m44s   role=website

--------------------

<details>
<summary>
Senario 1-
To meet the Security requirements, create a default policy for the database, website, and backup-system namespaces to deny all egress as a default. Follow the below naming convention for policies:

Policy name: default-{namespace-name}-network-policy
</summary>

```bash
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-website-network-policy
  namespace: website
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-database-network-policy
  namespace: database
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-backup-network-policy
  namespace: backup-system
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ```
  </details>
  
----

<details>
<summary>
Senario 2-
Since the website will need to get data from the database, create a network policy with the following details:

Policy Name: allow-website-ingress-to-database
Policy action: Allow database pod to get ingress communication from website pod on port 3306
</summary>

```bash
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: website-ingress
  namespace: website
spec:
  podSelector:
    matchLabels:
      role: website
  policyTypes:
  - Ingress
  ingress:
  - from:
      - ipBlock:
          cidr: 0.0.0.0/0
    ports:
      - protocol: TCP
        port: 80

  ```
</details>


<details>
<summary>
Senario 3-
The backup server will also need access to the database pod in order to take backups.

Create another policy similar to the website to database ingress that allows the backup server to communicate with database with the following details:

Policy Name: allow-backup-ingress-to-database
Policy action: Allow database pod to get ingress communication from backup pod on port 3306
</summary>
```bash


  ```
</details>
  
 <details>
 <summary>
Senario 4-
Modify the default network policy default-backup-network-policy for the backup-system namespace to allow the backup server egress to the NFS server with IP 10.1.2.3 on port 2049.
</summary>
```bash


  ```
</details>







