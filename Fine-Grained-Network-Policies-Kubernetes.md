### Fine-Grained Network Policies in Kubernetes: Building on Top of a Default Deny Policy
In the previous lab, Establishing a Baseline Security Posture with Default Deny Network Policies, we implemented a default deny policy in our Kubernetes cluster as a baseline security measure. This policy, applied to a three-tier application (comprising frontend, middleware, and MySQL database components), effectively blocked all communication between pods, acting as a protective barrier against unauthorized network interactions.

While such a comprehensive block ensures security, it also impedes necessary pod interactions vital for the functionality of our application. In this follow-up lab, we will focus on how to create granular network policies that allow the necessary inter-pod communication while preserving the overall security of the Kubernetes cluster.

Building on Default Deny
The default deny policy serves as a fundamental security baseline, rejecting any network interactions that aren't explicitly allowed. In effect, it's a stringent blacklist of all communications within the cluster. With this in place, we can now layer more specific network policies on top to define and allow the required interactions.

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress: []
  ```


Allowing Traffic between Specific Pods
For our three-tier application, the frontend needs to connect to the middleware, and the middleware needs to connect to the MySQL database. However, the frontend shouldn't have direct access to the MySQL database. We can shape these connections using specific network policies.

The YAML file for the network policy that allows traffic from the frontend to the middleware could look like this:
```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-middleware
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: middleware
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
  ```

This policy selects the middleware pod with the label "app: middleware" and allows traffic from pods with the label "app: frontend." It effectively establishes a connection from the frontend to the middleware.

Similarly, we can define a network policy to allow traffic from the middleware to the MySQL database.

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: middleware-to-mysql
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: mysql
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: middleware
  ```

Conclusion
With these specific network policies in place, our three-tier application will function correctly, as traffic is allowed where necessary. However, our Kubernetes cluster remains secure, as the default deny policy continues to block any other communications not explicitly permitted.

Remember, Kubernetes Network Policies apply at the namespace level. The policies we've created apply to the 'default' namespace, but in a multi-namespace cluster, similar policies would need to be created for each namespace. In upcoming labs, we'll explore more complex scenarios involving network policies in Kubernetes. Stay tuned!


### Namespace-Based Isolation with Kubernetes Network Policies
In Kubernetes, namespaces are a fundamental concept that enables logical partitioning of resources within a cluster. By dividing cluster resources among multiple namespaces, teams can segregate their environments, control access, and manage resources more effectively. As part of this multi-tenant approach, Kubernetes Network Policies play a crucial role. They allow administrators to define rules for traffic flowing between pods, even when these pods are spread across multiple namespaces.

In this lab, we'll explore namespace-based isolation with Kubernetes Network Policies, using a three-tier application deployed across different namespaces as an example. We'll consider several real-world scenarios, each highlighting a different aspect of network policy management.

nameSpace Selector
 we've used podSelector for managing traffic between pods within a namespace. For cross-namespace scenarios, we employ namespaceSelector, which, based on namespace labels, manages traffic between different namespaces. It offers a higher-level control for implementing network isolation at the namespace level.

Here's a simple YAML snippet using namespaceSelector:
```bash
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        project: myproject
  ```
In this example, the policy allows traffic from any pod in any namespace labeled with project: myproject.

Scenario 1: Denying All Traffic from Other Namespaces
Consider a production environment, where all three tiers of our application - frontend, middleware, and mysql - are deployed within a production namespace. For security reasons, it's crucial that pods within this namespace are not accessible from any other namespace. We can achieve this with the same default deny policy in the production namespace.

 the Network Policy would deny access to the pods in the production namespace from other namespaces such as qa and default.


Scenario 2: Allowing Traffic from a Specific Namespace
In another scenario, you might need to permit access from a specific namespace. For example, let's say you have a middleware pod in the pre-prod namespace that requires read access to the MySQL database in the production namespace.

To do this, we would use a namespaceSelector (instead of the podSelector) in the ingress rule that would allow traffic from all the pods of a specific namespace.

 we are selecting the entire pre-prod namespace in the ingress rule.

This would mean that that the middleware pod, along with any other existing or future pods that are deployed in the pre-prod namespace would be able to connect to the mysql pod in the production namespace.

Scenario 3: Allowing Traffic to Specific Pods from All Namespaces
In some cases, you might want to allow traffic from all namespaces to certain pods within a namespace. For instance, the frontend pods in our production namespace might need to be accessible from pods across all namespaces.

In this case, we have 4 different namespaces in our cluster. The pods belonging to the frontend-deploy deployment need to be accessible from all pods in the staging, default, logging as well as the production namespace.

To do this, we will again use the namespaceSelector but this time, we will use a blank rule to select all namespaces in the cluster.

Scenario 4: Allowing Traffic from Specific Pods in Another Namespace
Let's say you have a logging namespace with a log-aggregator pod that needs to access the middleware and MySQL pods in the production namespace to collect logs. Only the log-aggregator pod should be able to connect to MySQL, all other pods in the logging namespace be restricted.

To do this, we can combine namespaceSelector and the podSelector in the ingress rules to target specific pods in specific namespaces.

```bash
spec:
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
           namespace : production 
      podSelector:
        matchLabels:
           project: myproject
  ```

With Kubernetes Network Policies, we can create a secure and scalable multi-tenant environment. These different scenarios demonstrate how granular control over traffic can be achieved, offering versatile options to meet varying requirements of namespace-based isolation in Kubernetes deployments.

<details>
<summary>
Senario -
We have the 3-tier app deployed in the production namespace.

Carry out the following tasks to secure the pods:

Deploy an deny-all-ingress that blacklists all traffic in the cluster.
Allow ingress traffic from frontend pods to middleware
Allow ingress traffic from middleware to the mysql pod

Note: Do update the existing pods that are deployed in this cluster. Use the same labels/selectors that are currently configured.
</summary>

```bash
## deny-all-ingress in the production namspece.

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress

## To allow access from frontend to middleware.

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-middleware
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: middleware
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend

## and to allow access from middleware to mysql:

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-middleware-to-mysql
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: mysql
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: middleware

  ```
  </details>

  
<details>
<summary>
Senario -
We have created two new namespaces called alpha-prod and alpha-logger.

The alpha-logger namespace hosts several pods that are used for centralized logging of applications in this Kubernetes cluster.

A default deny policy has been applied on the alpha-prod namespace that denies all ingress traffic to the pods of this namespace.

Create a new, single network policy called allow-alpha-logger that would allow all pods from the alpha-logger namespace to the alpha-prod namespace.


Use the label function=logging when creating the network policy.
Do not alter the existing objects in anyway!
</summary>

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-alpha-logger
  namespace: alpha-prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              function: logging

  ```
  </details>

  
<details>
<summary>
Senario -
We have created two other new namespaces called beta-prod and beta-logger.

The beta-logger namespace hosts several pods that are used for centralized logging of applications in this Kubernetes cluster.

A default deny policy has been applied on the beta-prod namespace that denies all ingress traffic to the pods of this namespace.

Create a new, single network policy called allow-beta-logger-1 that would allows pods with label role=logger-1 from only the beta-logger namespace to the beta-prod namespace.


Use the label function=logging with the namespace selector when creating the network policy.
Do not alter the existing objects in anyway!
</summary>

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-beta-logger-1
  namespace: beta-prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              function: logging
        - podSelector:
            matchLabels:
              role: logger-1

  ```
  </details>


communication is a two-way street, and just as it's important to control who or what can access our pods, it's equally critical to regulate where our pods can send data. This is where egress network policies come into play.

You might wonder, if we have already defined a default deny and specific ingress policies, why would we need to define egress policies? After all, haven't we already set a solid defense against unwarranted inbound traffic?

While that's true, there's another side to the security equation. Egress policies add another layer of defense by ensuring our pods don't interact with services they aren't supposed to. For instance, you might not want your frontend pod to have direct access to a database pod or an external API. With egress policies, you can effectively control outbound traffic and prevent accidental data leaks or exposures.

A Comparative Look at Egress Vs. Ingress Network Policies
At first glance, egress network policies might look similar to ingress ones, but a key difference lies in their respective fields. For example, consider the following egress network policy for our frontend pod:
```bash
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: frontend-egress
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: middleware

  ```

In the above YAML snippet, instead of the ingress: field we used before, we have egress: to control outbound traffic. Likewise, under the policyTypes: field, we specify Egress instead of Ingress. Except for these changes, the rest of the structure closely mirrors the ingress policy we've been using thus far.

Scenario 1: Default Deny All Egress Traffic
For creating a default deny all egress policy, we specify an empty egress: [] array:
```bash
egress: []

  ```

Scenario 2: Egress to Specific Pods
If you need to allow egress to specific pods, you would need to include a podSelector under egress. The snippet might look like this:
```bash
egress:
- to:
  - podSelector:
      matchLabels:
        role: database

  ```

In the above snippet, egress is allowed to all pods labeled with "role: database".

Scenario 3: Egress to Specific Ports
To allow egress traffic to specific ports, you can specify ports under egress. The snippet might look like this:
```bash
egress:
- to:
  - podSelector:
      matchLabels:
        role: database
  ports:
  - protocol: TCP
    port: 3306

  ```
In this snippet, egress traffic is allowed to all pods labeled "role: database" on TCP port 3306.

Egress policy type

Egress policies are the policies that are used to control the traffic that leaves the pod. It helps to limit or allow the outbound traffic from the pods.
Egress rules are defined in the network policy spec section using the egress field.
 ```bash
spec:
  policyTypes:
  - Egress
  egress:
  - to:
    <rules>
  ```

<details>
<summary>
Senario -
The manifest will allow the traffic to a specific namespace.

</summary>

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-egress-to-ns
spec:
  podSelector: {} 
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: gamma

  ```
  </details>

  
<details>
<summary>
Senario -
The network policy deny-egress-to-ns is deployed in the default namespace that will allow egress traffic only from the default namespace to the gamma namespace.
But we don't need to access all pods in the gamma namespace. So, edit the deny-egress-to-ns policy to only allow egress traffic to pods with labels access: admin in gamma namespace.
</summary>

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-egress-to-ns
spec:
  podSelector: {} 
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: gamma
          podSelector:
            matchLabels:
              access: admin

controlplane ~ ➜  k describe netpol
Name:         deny-egress-to-ns
Namespace:    default
Created on:   2024-12-30 14:35:34 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Not affecting ingress traffic
  Allowing egress traffic:
    To Port: <any> (traffic allowed to all ports)
    To:
      NamespaceSelector: kubernetes.io/metadata.name=gamma
      PodSelector: access=admin
  Policy Types: Egress

## this is wrong ans as it will check egress condition as or insted of and  

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-egress-to-ns
spec:
  podSelector: {} 
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: gamma
        - podSelector:
            matchLabels:
              access: admin


controlplane ~ ➜  k describe netpol
Name:         deny-egress-to-ns
Namespace:    default
Created on:   2024-12-30 14:35:34 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Not affecting ingress traffic
  Allowing egress traffic:
    To Port: <any> (traffic allowed to all ports)
    To:
      NamespaceSelector: kubernetes.io/metadata.name=gamma
    To:
      PodSelector: access=admin
  Policy Types: Egress
  ```
  </details>


  
<details>
<summary>
Senario -

The policy we earlier applied will deny all the egress traffic except for some pods in the gamma namespace.
But we need the port 53 connectivity for all pods in the default namespace. So, create a new network policy egress-on-p53 that will allow the egress connection on port 53 in the default namespace.
Note: Allow using both TCP and UDP
</summary>

```bash


apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-on-p53
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - ports:
         - port: 53
           protocol: TCP
         - port: 53
           protocol: UDP

Name:         egress-on-p53
Namespace:    default
Created on:   2024-12-30 14:49:27 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Not affecting ingress traffic
  Allowing egress traffic:
    To Port: 53/TCP
    To Port: 53/UDP
    To: <any> (traffic not restricted by destination)
  Policy Types: Egress


  ```
  </details>



<details>
<summary>
Senario -

Kanoha Corp is using Kubernetes clusters to host their microservices-based applications. They have different namespaces in their clusters for separate teams and applications for security and isolation.
One of their applications, called ramen, is deployed in genin namespace, and it requires communication with another application called chakra that is deployed in jonin namespace. Also an administrative application hokage is deployed in the default namespace.
However, they want to enforce strict network policies to get add a level of security to their application. To implement policies, shikamaru, who is a Kubernetes administrator at kanoha, needs your help with the next set of questions.

Help shikamaru to create a network policy jonin-to-genin that will allow applications in jonin namespace to only have outbound traffic to ramen pods in genin namespace.
</summary>

```bash
kubectl -n genin get pods --show-labels

........
NAME    READY   STATUS    RESTARTS   AGE     LABELS
ramen   1/1     Running   0          9m10s   rank=genin



apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: jonin-to-genin
  namespace: jonin 
spec:
  podSelector: {} 
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: genin 
          podSelector:
            matchLabels:
              rank: genin


controlplane ~ ➜  k describe netpol -n jonin
Name:         jonin-to-genin
Namespace:    jonin
Created on:   2024-12-30 14:56:38 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Not affecting ingress traffic
  Allowing egress traffic:
    To Port: <any> (traffic allowed to all ports)
    To:
      NamespaceSelector: kubernetes.io/metadata.name=genin
      PodSelector: rank=genin
  Policy Types: Egress

  ```
  </details>



<details>
<summary>
Senario -
Konoha Corp wants applications in genin namespace should not have any egress connectivity. Create a network policy genin-no-egress for the same.

</summary>

```bash

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: 
 name : genin-no-egress
 namespace: genin
spec:
  policyTypes:
    - Egress
  egress : []
  podSelector: {}

  controlplane ~ ✖ k describe netpol genin-no-egress -n genin
Name:         genin-no-egress
Namespace:    genin
Created on:   2025-01-01 12:59:02 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Not affecting ingress traffic
  Allowing egress traffic:
    <none> (Selected pods are isolated for egress connectivity)
  Policy Types: Egress


  ```
  </details>



<details>
<summary>
Senario -
n administrative application called hokage is deployed in the default namespace and this application needs to have egress connectivity to both the ramen and chakra applications.
Create a network policy hokage-allow that will allow egress traffic from hokage pods to ramen and chakra pods in genin and jonin namespaces respectively.

Note: Make use of the pod labels.

</summary>

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: hokage-allow
spec:
  podSelector: 
   matchLabels:
      rank: hokage
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: genin 
          podSelector:
            matchLabels:
              rank: genin

        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: jonin
          podSelector:
            matchLabels:
              rank: jonin



controlplane ~ ➜  k describe netpol
Name:         hokage-allow
Namespace:    default
Created on:   2025-01-01 13:08:57 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     rank=hokage
  Not affecting ingress traffic
  Allowing egress traffic:
    To Port: <any> (traffic allowed to all ports)
    To:
      NamespaceSelector: kubernetes.io/metadata.name=genin
      PodSelector: rank=genin
    To:
      NamespaceSelector: kubernetes.io/metadata.name=jonin
      PodSelector: rank=jonin
  Policy Types: Egress

  ```
  </details>



<details>
<summary>
Senario -


</summary>

```bash


  ```
  </details>
