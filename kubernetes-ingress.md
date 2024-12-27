Up until now, you have been using NodePort service type to allow for external access to the website users at Company X.
This has been fine for initial testing, but your company will need to have an Ingress Controller installed so that you can take advantage of an Ingress instead of a service.
In this lab, you will go through the process of installing Traefik Ingress Controller as well as configure your Website to use an Ingress resource instead of a Service.
```bash
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
      app: flaskapp
  template:
    metadata:
      labels:
        app: flaskapp
    spec:
      containers:
      - name: flaskapp
        image: wbassler/flask-app-example:v0.0.1
        imagePullPolicy: Always
        ports:
        - containerPort: 5000
        env:
        - name: MYSQL_HOST
          value: "mysql-service.database.svc.cluster.local"
        - name: MYSQL_USER
          value: "root"
        - name: MYSQL_PASSWORD
          value: PASSWORD123
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
    app: flaskapp
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
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: PASSWORD123
        - name: MYSQL_DATABASE
          value: "userdb"
        ports:
        - containerPort: 3306
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
    app: mysql
  ```

<details>
<summary>
Senario -
add the Traefik Helm repo.
Install Traefik using helm with default values.

</summary>

```bash
root@controlplane ~ ➜  helm repo add traefik https://traefik.github.io/charts
"traefik" has been added to your repositories

root@controlplane ~ ➜  helm repo list
NAME    URL                             
traefik https://traefik.github.io/charts

root@controlplane ~ ✖ helm install traefik traefik/traefik
NAME: traefik
LAST DEPLOYED: Thu Dec 26 14:03:07 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
traefik with docker.io/traefik:v3.2.2 has been deployed successfully on default namespace !
  ```
</details>



<details>
<summary>
Senario -
Modify the values of the values.yaml file to utilize a NodePort service type and to use a NodePort for web and websecure ports. Just keep the default port numbers. Redeploy Traefik using Helm and the values file.
```bash
#values.yaml 
ports:
 traefik:
   port: 9000
   # -- Use hostPort if set.
   # hostPort: 9000
   #
   # -- Use hostIP if set. If not set, Kubernetes will default to 0.0.0.0, which
   # means it's listening on all your interfaces and all your IPs. You may want
   # to set this value if you need traefik to listen on specific interface
   # only.
   # hostIP: 192.168.100.10


   # Defines whether the port is exposed if service.type is LoadBalancer or
   # NodePort.
   #
   # -- You SHOULD NOT expose the traefik port on production deployments.
   # If you want to access it from outside your cluster,
   # use `kubectl port-forward` or create a secure ingress
   expose:
     default: false
   # -- The exposed port for this service
   exposedPort: 9000
   # -- The port protocol (TCP/UDP)
   protocol: TCP
 web:
   ## -- Enable this entrypoint as a default entrypoint. When a service doesn't explicitly set an entrypoint it will only use this entrypoint.
   # asDefault: true
   port: 8000
   # hostPort: 8000
   # containerPort: 8000
   expose:
     default: true
   exposedPort: 80
   ## -- Different target traefik port on the cluster, useful for IP type LB
   # targetPort: 80
   # The port protocol (TCP/UDP)
   protocol: TCP
   # -- Use nodeport if set. This is useful if you have configured Traefik in a
   # LoadBalancer.
   # nodePort: 32080
   # Port Redirections
   # Added in 2.2, you can make permanent redirects via entrypoints.
   # https://docs.traefik.io/routing/entrypoints/#redirection
   # redirectTo:
   #   port: websecure
   #   (Optional)
   #   priority: 10
   #
   # Trust forwarded  headers information (X-Forwarded-*).
   # forwardedHeaders:
   #   trustedIPs: []
   #   insecure: false
   #
   # Enable the Proxy Protocol header parsing for the entry point
   # proxyProtocol:
   #   trustedIPs: []
   #   insecure: false
 websecure:
   ## -- Enable this entrypoint as a default entrypoint. When a service doesn't explicitly set an entrypoint it will only use this entrypoint.
   # asDefault: true
   port: 8443
   # hostPort: 8443
   # containerPort: 8443
   expose:
     default: true
   exposedPort: 443
   ## -- Different target traefik port on the cluster, useful for IP type LB
   # targetPort: 80
   ## -- The port protocol (TCP/UDP)
   protocol: TCP
   # nodePort: 32443
   ## -- Specify an application protocol. This may be used as a hint for a Layer 7 load balancer.
   # appProtocol: https
   #
   ## -- Enable HTTP/3 on the entrypoint
   ## Enabling it will also enable http3 experimental feature
   ## https://doc.traefik.io/traefik/routing/entrypoints/#http3
   ## There are known limitations when trying to listen on same ports for
   ## TCP & UDP (Http3). There is a workaround in this chart using dual Service.
   ## https://github.com/kubernetes/kubernetes/issues/47249#issuecomment-587960741


service:
 type: LoadBalancer

  ```
</summary>

```bash
root@controlplane ~ ➜  cat values.yaml 
ports:
 traefik:
   port: 9000
   # -- Use hostPort if set.
   # hostPort: 9000
   #
   # -- Use hostIP if set. If not set, Kubernetes will default to 0.0.0.0, which
   # means it's listening on all your interfaces and all your IPs. You may want
   # to set this value if you need traefik to listen on specific interface
   # only.
   # hostIP: 192.168.100.10


   # Defines whether the port is exposed if service.type is LoadBalancer or
   # NodePort.
   #
   # -- You SHOULD NOT expose the traefik port on production deployments.
   # If you want to access it from outside your cluster,
   # use `kubectl port-forward` or create a secure ingress
   expose:
     default: false
   # -- The exposed port for this service
   exposedPort: 9000
   # -- The port protocol (TCP/UDP)
   protocol: TCP
 web:
   ## -- Enable this entrypoint as a default entrypoint. When a service doesn't explicitly set an entrypoint it will only use this entrypoint.
   # asDefault: true
   port: 8000
   # hostPort: 8000
   # containerPort: 8000
   expose:
     default: true
   exposedPort: 80
   ## -- Different target traefik port on the cluster, useful for IP type LB
   # targetPort: 80
   # The port protocol (TCP/UDP)
   protocol: TCP
   # -- Use nodeport if set. This is useful if you have configured Traefik in a
   # LoadBalancer.
   nodePort: 32080
   # Port Redirections
   # Added in 2.2, you can make permanent redirects via entrypoints.
   # https://docs.traefik.io/routing/entrypoints/#redirection
   # redirectTo:
   #   port: websecure
   #   (Optional)
   #   priority: 10
   #
   # Trust forwarded  headers information (X-Forwarded-*).
   # forwardedHeaders:
   #   trustedIPs: []
   #   insecure: false
   #
   # Enable the Proxy Protocol header parsing for the entry point
   # proxyProtocol:
   #   trustedIPs: []
   #   insecure: false
 websecure:
   ## -- Enable this entrypoint as a default entrypoint. When a service doesn't explicitly set an entrypoint it will only use this entrypoint.
   # asDefault: true
   port: 8443
   # hostPort: 8443
   # containerPort: 8443
   expose:
     default: true
   exposedPort: 443
   ## -- Different target traefik port on the cluster, useful for IP type LB
   # targetPort: 80
   ## -- The port protocol (TCP/UDP)
   protocol: TCP
   nodePort: 32443
   ## -- Specify an application protocol. This may be used as a hint for a Layer 7 load balancer.
   # appProtocol: https
   #
   ## -- Enable HTTP/3 on the entrypoint
   ## Enabling it will also enable http3 experimental feature
   ## https://doc.traefik.io/traefik/routing/entrypoints/#http3
   ## There are known limitations when trying to listen on same ports for
   ## TCP & UDP (Http3). There is a workaround in this chart using dual Service.
   ## https://github.com/kubernetes/kubernetes/issues/47249#issuecomment-587960741


service:
 type: NodePort


root@controlplane ~ ➜  helm upgrade --values=./values.yaml traefik traefik/traefik
Release "traefik" has been upgraded. Happy Helming!
NAME: traefik
LAST DEPLOYED: Thu Dec 26 14:10:22 2024
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
traefik with docker.io/traefik:v3.2.2 has been deployed successfully on default namespace !


root@controlplane ~ ➜  helm list
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS      CHART            APP VERSION
traefik default         2               2024-12-26 14:10:22.975954268 +0000 UTC deployed    traefik-33.2.1   v3.2.2     


root@controlplane ~ ✖ k get all
NAME                           READY   STATUS    RESTARTS   AGE
pod/traefik-6bbb7d95b7-b9c8r   1/1     Running   0          93s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP                      158m
service/traefik      NodePort    10.104.220.6   <none>        80:32080/TCP,443:32443/TCP   8m48s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/traefik   1/1     1            1           8m48s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/traefik-6bbb7d95b7   1         1         1       93s
replicaset.apps/traefik-75b955cf7    0         0         0       8m48s

  ```
</details>



<details>
<summary>
Senario -
Create a new service for the website deployment called website-service. Its ports should be named http and have both a targetport and port of 5000. Be sure to include the selector as app: flaskapp.
</summary>

```bash
cat svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: website-service
  namespace: website
spec:
  ports:
    - port: 5000
      targetPort: 5000
      name: http
  selector:
    app: flaskapp

    
root@controlplane ~ ➜  k apply -f svc.yaml 
service/website-service created

root@controlplane ~ ➜  k describe svc -n website
Name:              website-service
Namespace:         website
Labels:            <none>
Annotations:       <none>
Selector:          app=flaskapp
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.108.37.220
IPs:               10.108.37.220
Port:              http  5000/TCP
TargetPort:        5000/TCP
Endpoints:         10.50.0.4:5000,10.50.192.1:5000,10.50.192.2:5000
Session Affinity:  None
Events:            <none>
  ```
</details>



<details>
<summary>
Senario -
Delete the old NodePort website-service-nodeport service. We will be creating an Ingress to replace it for external connections.

</summary>

```bash
 kubectl delete -n website svc website-service-nodeport
service "website-service-nodeport" deleted

root@controlplane ~ ➜  k get svc -n website
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
website-service   ClusterIP   10.108.37.220   <none>        5000/TCP   2m27s
  ```
</details>



<details>
<summary>
Senario -
Create an Ingress for the website called website-ingress in website namespace.

Create a default backend to the website-service on the port name of http. The rules should not include a host value, but should include an http path of/, with a backend to the website-service on port name of http.

To verify the Ingress in your cluster and ensure accessibility to CompanyX's website and confirm its availability.
</summary>

```bash
root@controlplane ~ ➜  cat ingress.yaml 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: website-ingress
  namespace: website
spec:
  defaultBackend:
    service:
      name: website-service
      port:
        name: http
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: website-service
                port:
                  name: http 

root@controlplane ~ ➜ k apply -f ingress.yaml 
ingress.networking.k8s.io/website-ingress created

root@controlplane ~ ➜  k get ingress -n website
NAME              CLASS     HOSTS   ADDRESS   PORTS   AGE
website-ingress   traefik   *                 80      24s


root@controlplane ~ ➜  k describe ingress -n website
Name:             website-ingress
Labels:           <none>
Namespace:        website
Address:          
Ingress Class:    traefik
Default backend:  website-service:http (10.50.0.4:5000,10.50.192.1:5000,10.50.192.2:5000)
Rules:
  Host        Path  Backends
  ----        ----  --------
  *           
              /   website-service:http (10.50.0.4:5000,10.50.192.1:5000,10.50.192.2:5000)
Annotations:  <none>
Events:       <none>




root@controlplane ~ ➜  curl localhost:32080

                                  
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
                    font-family: Arial, sans-serif;
                }

                .header {
                    background-color: #f2f2f2;
                    text-align: center;
                    padding: 20px;
                }

                .login-container {
                    position: absolute;
                    left: 50%;
                    top: 50%;
                    transform: translate(-50%, -50%);
                    padding: 20px;
                    background: #fff;
                    box-shadow: 0 2px 4px rgba(0,0,0,0.2);
                    width: 300px;
                    border-radius: 5px;
                }

                input[type=text], input[type=password] {
                    width: 100%;
                    padding: 15px;
                    margin: 5px 0 20px 0;
                    display: inline-block;
                    border: 1px solid #ccc;
                    box-sizing: border-box;
                }

                button {
                    background-color: #4CAF50;
                    color: white;
                    padding: 14px 20px;
                    margin: 8px 0;
                    border: none;
                    cursor: pointer;
                    width: 100%;
                }

                button:hover {
                    opacity: 0.8;
                }
            </style>
        </head>
        <body>

        <div class="header">
            <h2>Welcome to Company X!</h2>
        </div>

        <div class="login-container">
            <form action="/auth" method="post">
                <label for="username">Username</label>
                <input type="text" placeholder="Enter Username" name="username" required>

                <label for="password">Password</label>
                <input type="password" placeholder="Enter Password" name="password" required>

                <button type="submit">Login</button>
            </form>
        </div>

        </body>
        </html>

  ```
</details>


In this lab, we will focus on securing the Ingress for Company X's website using Cert-manager and Let's Encrypt.
Company X needs to ensure that all data transmitted between their users and their web servers is encrypted to protect sensitive information such as personal details and payment information.
By securing their Ingress, Company X can guarantee that communications are secure, thus enhancing customer trust and compliance with data protection regulations.

Here is a list of resources that are created for you as part of this lab:

Namespaces

traefik
website
database
Deployments

website-deployment in the website namespace
database-deployment in the database namespace
Services

website-service-nodeport in the website namespace (type: NodePort)
mysql-service in the database namespace (type: ClusterIP, headless)
NetworkPolicy

default-deny-egress in the database namespace
ConfigMap

mysql-init-db in the database namespace
Ingress

website-ingress in the website namespace

```bash
## install 

apiVersion: v1
kind: Namespace
metadata:
  name: traefik
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
      app: website
  template:
    metadata:
      labels:
        app: website
    spec:
      containers:
      - name: flaskapp
        image: wbassler/flask-app-example:v0.0.1
        imagePullPolicy: Always
        ports:
        - containerPort: 5000
        env:
        - name: MYSQL_HOST
          value: "mysql-service.database.svc.cluster.local"
        - name: MYSQL_USER
          value: "root"
        - name: MYSQL_PASSWORD
          value: PASSWORD123
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
    app: website
---
apiVersion: v1
kind: Namespace
metadata:
  name: database
  labels:
    role: database
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: database
spec:
  podSelector: {}
  policyTypes:
  - Egress
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
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: PASSWORD123
        - name: MYSQL_DATABASE
          value: "userdb"
        ports:
        - containerPort: 3306
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
    app: mysql
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: website-ingress
  namespace: website
spec:
  defaultBackend:
    service:
      name: website-service-nodeport
      port:
        name: http
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: website-service-nodeport
            port:
              name: http


cat values.yaml 

logs:
 general:
   level: INFO
 access:
   enabled: true
   filters: {}
   addInternals:
   fields:
     general:
       defaultmode: keep
       names: {}
     headers:
       defaultmode: drop
       names: {}
ports:
 traefik:
   port: 9000
   expose:
     default: false
   exposedPort: 9000
   protocol: TCP
 web:
   port: 8000
   expose:
     default: true
   exposedPort: 80
   protocol: TCP
   nodePort: 32080
 websecure:
   port: 8443
   expose:
     default: true
   exposedPort: 443
   protocol: TCP
   nodePort: 32443
   http3:
     enabled: false
   tls:
     enabled: true
     options: ""
     certResolver: ""
     domains: []
 metrics:
   port: 9100
   expose:
     default: false
  ```

<details>
<summary>
Senario -
To get started, add the Cert-manager repo on helm in your terminal.

Use the following artifactory link to get the instructions.
https://artifacthub.io/packages/helm/cert-manager/cert-manager?modal=install

Create a namespace called cert-manager for Cert-manager.

Install Cert-manager in the newly created cert-manager namespace using Helm, including all necessary Custom Resource Definitions (CRDs).
Note: Installation will take sometime.

</summary>

```bash

root@controlplane ~ ➜  helm repo add cert-manager https://charts.jetstack.io
"cert-manager" has been added to your repositories

root@controlplane ~ ✖ helm repo add jetstack https://charts.jetstack.io
"jetstack" has been added to your repositories


root@controlplane ~ ➜  helm repo list
NAME            URL                       
cert-manager    https://charts.jetstack.io



root@controlplane ~ ➜  k create ns cert-manager
namespace/cert-manager created


root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  helm install cert-manager jetstack/cert-manager  --namespace cert-manager--set installCRDs=true
NAME: cert-manager
LAST DEPLOYED: Thu Dec 26 16:32:22 2024
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
⚠️  WARNING: `installCRDs` is deprecated, use `crds.enabled` instead.
cert-manager v1.16.2 has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://cert-manager.io/docs/configuration/

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://cert-manager.io/docs/usage/ingress/

  ```
</details>




<details>
<summary>
Senario -
Create an initial staging Issuer resource named letsencrypt-staging in the website namespace.

Use the Staging server URL: https://acme-staging-v02.api.letsencrypt.org/directory
Email address: user@gmail.com
Name the secret key: letsencrypt-staging
and configure an http01 challenge on the Ingress website-ingress in the website namespace.
</summary>

```bash
root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  cat cert.yaml 
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
  namespace: website
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: user@gmail.com
    privateKeySecretRef:
          name: letsencrypt-staging
    solvers:
    - http01:
            ingress:
              name: website-ingress

root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  k apply -f cert.yaml 
issuer.cert-manager.io/letsencrypt-staging created



root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  k get Issuer -n website
NAME                  READY   AGE
letsencrypt-staging   True    3m14s

root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  k describe Issuer -n website
Name:         letsencrypt-staging
Namespace:    website
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Issuer
Metadata:
  Creation Timestamp:  2024-12-26T16:35:12Z
  Generation:          1
  Resource Version:    20498
  UID:                 886d1ad1-56dc-477e-be20-98ff05d2e969
Spec:
  Acme:
    Email:  user@gmail.com
    Private Key Secret Ref:
      Name:  letsencrypt-staging
    Server:  https://acme-staging-v02.api.letsencrypt.org/directory
    Solvers:
      http01:
        Ingress:
          Name:  website-ingress
Status:
  Acme:
    Last Private Key Hash:  bKZRjdwBw2Ooq4rUkMMAKB/KCyySjgg3DYUxuWI9fOM=
    Last Registered Email:  user@gmail.com
    Uri:                    https://acme-staging-v02.api.letsencrypt.org/acme/acct/177706034
  Conditions:
    Last Transition Time:  2024-12-26T16:35:12Z
    Message:               The ACME account was registered with the ACME server
    Observed Generation:   1
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>

  ```
</details>


<details>
<summary>
Senario -
Now that we have a staging issuer, update the Ingress website-ingress to use the staging issuer for the domain companyx-website.com, and specify secret name as web-ssl.
Ensure to add the necessary annotations and configure the TLS section with the host name and secret name.


</summary>

```bash


root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  cat ingress.yaml 

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: website-ingress
  namespace: website
  annotations:
    cert-manager.io/issuer: letsencrypt-staging
spec:
  tls:
  - hosts:
    - companyx-website.com
    secretName: web-ssl
  defaultBackend:
    service:
      name: website-service-nodeport
      port:
        name: http
  rules:
  - host: companyx-website.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: website-service-nodeport
            port:
              name: http

root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  k apply -f ingress.yaml 
ingress.networking.k8s.io/website-ingress configured



root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  k get ingress -n website
NAME              CLASS    HOSTS                  ADDRESS   PORTS     AGE
website-ingress   <none>   companyx-website.com             80, 443   26m


root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  
Name:             website-ingressis 📦 v1.16.2 via ⎈ v3.16.4 ➜  k describe ingress -n website 
Labels:           <none>
Namespace:        website
Address:          
Ingress Class:    <none>
Default backend:  website-service-nodeport:http (10.50.0.4:5000,10.50.192.1:5000,10.50.192.2:5000)
TLS:
  web-ssl terminates companyx-website.com
Rules:
  Host                  Path  Backends
  ----                  ----  --------
  companyx-website.com  
                        /.well-known/acme-challenge/OQ8qHCxnT5SM5tjtkOGkbt-Tpz_HPzX0ATcf5zZ4UuI   cm-acme-http-solver-6fg6d:8089 (10.0.1.73:8089)
                        /                                                                         website-service-nodeport:http (10.50.0.4:5000,10.50.192.1:5000,10.50.192.2:5000)
Annotations:            cert-manager.io/issuer: letsencrypt-staging
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  CreateCertificate  81s   cert-manager-ingress-shim  Successfully created Certificate "web-ssl"

  ```
</details>


<details>
<summary>
Senario -
Our Ingress has now been updated with Cert-manager, and a certificate has been created as a secret. We can also check the status of our certificate by describing the Certificate custom resource.
kubectl describe certificate -n website

```bash

root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  kubectl get certificate -n website
NAME      READY   SECRET    AGE
web-ssl   False   web-ssl   3m48s


root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  kubectl describe certificate -n website
Name:         web-ssl
Namespace:    website
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2024-12-26T16:37:55Z
  Generation:          1
  Owner References:
    API Version:           networking.k8s.io/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  website-ingress
    UID:                   a212b81d-bc9d-449f-abda-3bcfef0cb19e
  Resource Version:        20837
  UID:                     02c09844-cb54-4c2e-a2fb-c12e5c5ce6cb
Spec:
  Dns Names:
    companyx-website.com
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       Issuer
    Name:       letsencrypt-staging
  Secret Name:  web-ssl
  Usages:
    digital signature
    key encipherment
Status:
  Conditions:
    Last Transition Time:        2024-12-26T16:37:55Z
    Message:                     Issuing certificate as Secret does not exist
    Observed Generation:         1
    Reason:                      DoesNotExist
    Status:                      True
    Type:                        Issuing
    Last Transition Time:        2024-12-26T16:37:55Z
    Message:                     Issuing certificate as Secret does not exist
    Observed Generation:         1
    Reason:                      DoesNotExist
    Status:                      False
    Type:                        Ready
  Next Private Key Secret Name:  web-ssl-jh8p2
Events:
  Type    Reason     Age    From                                       Message
  ----    ------     ----   ----                                       -------
  Normal  Issuing    3m29s  cert-manager-certificates-trigger          Issuing certificate as Secret does not exist
  Normal  Generated  3m29s  cert-manager-certificates-key-manager      Stored new private key in temporary Secret resource "web-ssl-jh8p2"
  Normal  Requested  3m29s  cert-manager-certificates-request-manager  Created new CertificateRequest resource "web-ssl-1"

  ```
Here we see that our certificate was indeed issued, generated, and requested.


If the certificate was successfully created, proceed to create a new Production Issuer.
Create a production Issuer resource named letsencrypt-production in the website namespace.
Use the Staging server URL: https://acme-v02.api.letsencrypt.org/directory
Email address: user@gmail.com
Name the secret key: letsencrypt-production
and configure an http01 challenge on the Ingress website-ingress in the website namespace.
</summary>

```bash


root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  cat issuer.yaml 
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-production
  namespace: website
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: user@gmail.com
    privateKeySecretRef:
          name: letsencrypt-production
    solvers:
    - http01:
            ingress:
              name: website-ingress


root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  k get issuer -n website
NAME                     READY   AGE
letsencrypt-production   True    15s


root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  k describe   issuer -n website
Name:         letsencrypt-production
Namespace:    website
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Issuer
Metadata:
  Creation Timestamp:  2024-12-26T16:43:14Z
  Generation:          1
  Resource Version:    21532
  UID:                 0f123e7b-3702-44b3-912a-c1f97303a139
Spec:
  Acme:
    Email:  user@gmail.com
    Private Key Secret Ref:
      Name:  letsencrypt-production
    Server:  https://acme-v02.api.letsencrypt.org/directory
    Solvers:
      http01:
        Ingress:
          Name:  website-ingress
Status:
  Acme:
    Last Private Key Hash:  ym0TIXOTawPj0nXBnWNpemmRP/G1mfHq3vDCacgOW1M=
    Last Registered Email:  user@gmail.com
    Uri:                    https://acme-v02.api.letsencrypt.org/acme/acct/2135281935
  Conditions:
    Last Transition Time:  2024-12-26T16:43:14Z
    Message:               The ACME account was registered with the ACME server
    Observed Generation:   1
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>


  ```
</details>



<details>
<summary>
Senario -
Now that we have a tested staging and a have an production issuer, update the Ingress website-ingress to use the production issuer for the domain companyx-website.com.
Ensure to add the necessary annotations.
```bash


root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  k get ingress website-ingress
NAME              CLASS    HOSTS                  ADDRESS   PORTS     AGE
website-ingress   <none>   companyx-website.com             80, 443   34m


root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  k describe ingress website-ingress -n website
Name:             website-ingress
Labels:           <none>
Namespace:        website
Address:          
Ingress Class:    <none>
Default backend:  website-service-nodeport:http (10.50.0.4:5000,10.50.192.1:5000,10.50.192.2:5000)
TLS:
  web-ssl terminates companyx-website.com
Rules:
  Host                  Path  Backends
  ----                  ----  --------
  companyx-website.com  
                        /.well-known/acme-challenge/OQ8qHCxnT5SM5tjtkOGkbt-Tpz_HPzX0ATcf5zZ4UuI   cm-acme-http-solver-6fg6d:8089 (10.0.1.73:8089)
                        /                                                                         website-service-nodeport:http (10.50.0.4:5000,10.50.192.1:5000,10.50.192.2:5000)
Annotations:            cert-manager.io/issuer: letsencrypt-staging
Events:
  Type    Reason             Age    From                       Message
  ----    ------             ----   ----                       -------
  Normal  CreateCertificate  9m17s  cert-manager-ingress-shim  Successfully created Certificate "web-ssl"


  ```
</summary>

```bash

root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  kubectl annotate ingress -n website website-ingress cert-manager.io/issuer=letsencrypt-production --overwrite
ingress.networking.k8s.io/website-ingress annotated


root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  k get ingress website-ingress -n website
NAME              CLASS    HOSTS                  ADDRESS   PORTS     AGE
website-ingress   <none>   companyx-website.com             80, 443   35m


root@controlplane ~/cert-manager is 📦 v1.16.2 via ⎈ v3.16.4 ➜  k describe ingress website-inress ebsite
Name:             website-ingress
Labels:           <none>
Namespace:        website
Address:          
Ingress Class:    <none>
Default backend:  website-service-nodeport:http (10.50.0.4:5000,10.50.192.1:5000,10.50.192.2:5000)
TLS:
  web-ssl terminates companyx-website.com
Rules:
  Host                  Path  Backends
  ----                  ----  --------
  companyx-website.com  
                        /.well-known/acme-challenge/ZOWmYFDWpzWrTAXTcb8QIW0ob8tbQraGpvel_Y_g_TQ   cm-acme-http-solver-vj8cd:8089 (10.0.0.188:8089)
                        /                                                                         website-service-nodeport:http (10.50.0.4:5000,10.50.192.1:5000,10.50.192.2:5000)
Annotations:            cert-manager.io/issuer: letsencrypt-production
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  CreateCertificate  10m   cert-manager-ingress-shim  Successfully created Certificate "web-ssl"
  Normal  UpdateCertificate  12s   cert-manager-ingress-shim  Successfully updated Certificate "web-ssl"


  ```
</details>

Now that we have learned more about the benefits of using CNI network policies, we would like to implement them into our Cluster at CompanyX.

We shall going to go through our current cluster setup and modify the current network policies to become CiliumNetworkPolicy resources..
Here is the list of resources that were created for you as part of this lab:

Namespaces
backup-system
website
database
Deployments
website-deployment in the website namespace
database-deployment in the database namespace
Services
website-service-nodeport in the website namespace (type: NodePort)
mysql-service in the database namespace (type: ClusterIP, headless)
Network Policies
default-deny-egress in the database namespace (Deny all egress traffic)
allow-website-ingress-to-database in the database namespace (Allow ingress traffic from the website namespace to the database)
allow-backup-ingress-to-database in the database namespace (Allow ingress traffic from the backup-system namespace to the database)
allow-backup-to-backup-server in the backup-system namespace (Allow egress traffic from the backup system to a specific IP block)
ConfigMaps
mysql-init-db in the database namespace
Summary of Resources
Namespaces:
backup-system
website
database
Deployments:
website-deployment in the website namespace
database-deployment in the database namespace
Services:
website-service-nodeport in the website namespace
mysql-service in the database namespace
Network Policies:
default-deny-egress in the database namespace
allow-website-ingress-to-database in the database namespace
allow-backup-ingress-to-database in the database namespace
allow-backup-to-backup-server in the backup-system namespace
ConfigMaps:
mysql-init-db in the database namespace

```bash

root@controlplane ~ ➜  cat application.yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: backup-system
---
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
      - name: flaskapp
        image: wbassler/flask-app-example:v0.0.1
        imagePullPolicy: Always
        ports:
        - containerPort: 5000
        env:
        - name: MYSQL_HOST
          value: "mysql-service.database.svc.cluster.local"
        - name: MYSQL_USER
          value: "root"
        - name: MYSQL_PASSWORD
          value: PASSWORD123
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
    app: website
---
apiVersion: v1
kind: Namespace
metadata:
  name: database
  labels:
    app: database
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: database
spec:
  podSelector: {}
  policyTypes:
  - Egress
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
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: PASSWORD123
        - name: MYSQL_DATABASE
          value: "userdb"
        ports:
        - containerPort: 3306
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
    app: mysql
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-website-ingress-to-database
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          app: website
    - podSelector:
        matchLabels:
          app: website
    ports:
    - protocol: TCP
      port: 3306
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backup-ingress-to-database
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          app: backup-system
    - podSelector:
        matchLabels:
          app: backup-system
    ports:
    - protocol: TCP
      port: 3306
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backup-to-backup-server
  namespace: backup-system
spec:
  podSelector:
    matchLabels:
      app: backup-system
  egress:
  - to:
    - ipBlock:
        cidr: 10.1.2.3/32
    ports:
    - protocol: TCP
      port: 2049
  policyTypes:
  - Egress

  ```


<details>
<summary>
Senario -
Convert the network policy in the database namespace named allow-website-ingress-to-database to a new CiliumNetworkPolicy resource.
Name the new resource allow-website-ingress-to-database-cnp to allow communication between the website pods and the database.
And save the resource manifest at location /root/allow-website-ingress-to-database-cnp.yaml
```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-website-ingress-to-database
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          app: website
    - podSelector:
        matchLabels:
          app: website
    ports:
    - protocol: TCP
      port: 3306
  policyTypes:
  - Ingress

  ```

  Delete the existing allow-website-ingress-to-database policy and apply the new Cilium Network Policy 

</summary>

```bash
#allow-website-ingress-to-database-cnp.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-website-ingress-to-database-cnp
  namespace: database
spec:
  endpointSelector:
    matchLabels:
      app: database
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: website
    toPorts:
    - ports:
      - port: "3306"
        protocol: TCP



root@controlplane ~ ✖ k describe netpol  -n database

Name:         allow-website-ingress-to-database
Namespace:    database
Created on:   2024-12-26 16:59:00 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     app=database
  Allowing ingress traffic:
    To Port: 3306/TCP
    From:
      NamespaceSelector: app=website
    From:
      PodSelector: app=website
  Not affecting egress traffic
  Policy Types: Ingress

  ```
</details>


<details>
<summary>
Senario -
Convert the network policy in the backup-system namespace named allow-backup-to-backup-server to a new CiliumNetworkPolicy called allow-backup-to-backup-server-cnp.

This policy should allow egress from the backup system pods to the external backup server, maintaining the protocol and port to create a layer 4 type of policy.

And save the resource manifest at location /root/allow-backup-to-backup-server-cnp.yaml

```bash

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backup-to-backup-server
  namespace: backup-system
spec:
  podSelector:
    matchLabels:
      app: backup-system
  egress:
  - to:
    - ipBlock:
        cidr: 10.1.2.3/32
    ports:
    - protocol: TCP
      port: 2049
  policyTypes:
  - Egress
  ```
</summary>

```bash
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-backup-to-backup-server-cnp
  namespace: backup-system
spec:
  endpointSelector:
    matchLabels:
      app: backup-system
  egress:
  - toCIDR:
    - 10.1.2.3/32
    toPorts:
    - ports:
      - port: "2049"
        protocol: TCP


root@controlplane ~ ➜     kubectl delete networkpolicy allow-backup-to-backup-server -n backup-system
networkpolicy.networking.k8s.io "allow-backup-to-backup-server" deleted

root@controlplane ~ ➜     kubectl apply -f /root/allow-backup-to-backup-server-cnp.yaml
ciliumnetworkpolicy.cilium.io/allow-backup-to-backup-server-cnp created
  ```
</details>





<details>
<summary>
Senario -
Convert the network policy in the database namespace named allow-backup-ingress-to-database to a new CiliumNetworkPolicy called allow-backup-ingress-to-database-cnp.
This policy will allow the backup system pods to connect to the database and take a snapshot. This should be similar to the website ingress policy.
Delete the existing network policy named allow-backup-ingress-to-database in the database namespace and apply the new policy allow-backup-ingress-to-database-cnp.
```bash
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: allow-backup-ingress-to-database
     namespace: database
   spec:
     podSelector:
       matchLabels:
         app: database
     ingress:
     - from:
       - namespaceSelector:
           matchLabels:
             app: backup-system
       - podSelector:
           matchLabels:
             app: backup-system
       ports:
       - protocol: TCP
         port: 3306
     policyTypes:
     - Ingress

  ```

</summary>

```bash
  
root@controlplane ~ ➜     kubectl delete networkpolicy allow-backup-ingress-to-database -n database
networkpolicy.networking.k8s.io "allow-backup-ingress-to-database" deleted

root@controlplane ~ ➜  vi /root/allow-backup-ingress-to-database-cnp.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-backup-ingress-to-database-cnp
  namespace: database
spec:
  endpointSelector:
    matchLabels:
      app: database
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: backup-system
    toPorts:
    - ports:
      - port: "3306"
        protocol: TCP

root@controlplane ~ ➜     kubectl apply -f /root/allow-backup-ingress-to-database-cnp.yaml
ciliumnetworkpolicy.cilium.io/allow-backup-ingress-to-database-cnp created


  ```
</details>


<details>
<summary>
Senario -
Create a CiliumClusterwideNetworkPolicy named allow-ingress-to-website-api for the website pods in the website namespace.
This policy should allow pods with the label app: integration within the cluster to connect to the /api endpoint on port 80.
The rules should permit only POST requests and require an X-API-KEY header with the value integrationKey123.


</summary>

```bash

root@controlplane ~ ➜  vi  /root/allow-ingress-to-website-api.yaml

apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: allow-ingress-to-website-api
spec:
  endpointSelector:
    matchLabels:
      app: website
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: integration
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "POST"
          path: "/api"
          headers:
          - "X-API-KEY: integrationKey123"

          

root@controlplane ~ ➜     kubectl apply -f /root/allow-ingress-to-website-api.yaml
ciliumclusterwidenetworkpolicy.cilium.io/allow-ingress-to-website-api created

  ```
</details>



