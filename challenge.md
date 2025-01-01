The Final Challenge: Implementing Network Policies for Hogwarts
Welcome to the final challenge of our hands-on Kubernetes Network Policy course!

In this challenge, you'll take on the role of a Network Security Engineer for a fictitious company named "Hogwarts." Hogwarts has a unique business setup; it runs different magical departments, each represented by various namespaces in the Kubernetes cluster. Each namespace, be it charms, darkarts, or potions, is responsible for a different set of pods and services.

final-challenge

Recently, Hogwarts encountered some security incidents. The lack of proper isolation between applications running in different namespaces had exposed Hogwarts to potential security risks. On further examination, it was evident that the Kubernetes cluster's pods were not properly isolated. Without the right network policies, unrestricted communication across different departments had become a serious security concern.

Recognizing the need for stringent security measures, Hogwarts has now decided to implement network policies within their Kubernetes cluster. These policies will control and manage traffic flow at the IP address or port level. By defining precise ingress and egress rules, Hogwarts aims to restrict inter-department communication to only what's necessary, thus ensuring that each department can carry out its roles and responsibilities without posing a security risk to the others.

In this challenge, your task is to design and implement a set of network policies that will:

Isolate each department's applications from each other, providing a solid baseline of security.
Define specific ingress and egress rules for each department based on their requirements.
Limit traffic between pods to only the necessary pods/services in the other departments.
This challenge will test all the skills and knowledge you've acquired throughout this course. Remember, each department should only have access to the necessary resources from the other namespaces. The goal is to ensure communication is as restricted as possible without hindering Hogwarts' operational efficiency.

Take this final challenge as your opportunity to demonstrate your understanding of Kubernetes network policies and your ability to apply them in real-world scenarios. Good luck, and may the magic be with you!



<details>
<summary>
Senario -
Welcome to the final challenge of our hands-on Kubernetes Network Policy course! In this lab, you'll apply the knowledge and skills you've learned so far to secure the Hogwarts' Kubernetes cluster. Please carefully read the tasks below and use the provided hints to guide you.

The cluster has several kubernetes pods running in different namespaces. However, due to the default behaviour of the Network policies, all pods can access other pods.

Your task is to create network policies in the respective namespaces to acheive some level of security and traffic control inside the cluster.

The Pods and Namespaces details are given below.

Namespace	Pods
default	dumbledore
charms	expecto-patronum, lumos-and-nox, stupefy
darkarts	avada-kedavra, imperio, parseltongue
potions	polyjuice, veritaserum, wolfsbane
Task 1: Secure the "charms" Namespace
Your first task is to secure the charms namespace:

Create a network policy, charms-and-students, that allows ingress traffic from pods with a specific label within the charms namespace. (Hint: You'll need to use a pod selector inside the ingress rule)
Policy charms-and-students allows ingress traffic from pods with labels hogwarts: admitted


Create another network policy, charms-no-external-access, that restricts ingress and egress traffic within the charms namespace. (Hint: Use the policyTypes field to specify both Ingress and Egress)
Policy charms-no-egress denies all egress traffic to all namespaces except its own (charms namespace)
</summary>

```bash

controlplane ~ ➜  cat charms-and-students.yaml 

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name : charms-and-students
 namespace: charms
spec:
  policyTypes:
    - Ingress
  ingress: 
    - from:
      -  podSelector:
           matchLabels:
             hogwarts: admitted
  podSelector: {}

  controlplane ~ ➜  k describe netpol -n charms
Name:         charms-and-students
Namespace:    charms
Created on:   2025-01-01 13:50:54 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Allowing ingress traffic:
    To Port: <any> (traffic allowed to all ports)
    From:
      PodSelector: hogwarts=admitted
  Not affecting egress traffic
  Policy Types: Ingress



 cat charms-no-egress.yaml 

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name : charms-no-egress
 namespace: charms
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        -  namespaceSelector:
             matchLabels:
               kubernetes.io/metadata.name: charms

               
controlplane ~ ➜  k describe netpol -n charms

Name:         charms-no-egress
Namespace:    charms
Created on:   2025-01-01 13:56:36 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Not affecting ingress traffic
  Allowing egress traffic:
    To Port: <any> (traffic allowed to all ports)
    To:
      NamespaceSelector: kubernetes.io/metadata.name=charms
  Policy Types: Egress


  ```
  </details>


<details>
<summary>
Senario -
Task 2: Set Rules for "default" Namespace
In the default namespace, there's a pod called dumbledore.
Policy default-allow-all allows the dumbledore pod unrestricted ingress and egress traffic from/to all pods?
Create a network policy, default-allow-all, that allows all ingress and egress traffic from/to the dumbledore pod. (Hint: Don't specify any from or to rules in your policy)

</summary>

```bash

controlplane ~ ➜  cat default-allow-all.yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: 
 name : default-allow-all
spec:
  policyTypes:
    - Egress
    - Ingress
  ingress:
    - {}
  egress: 
    - {}
  podSelector: 
         matchLabels:
           i-am: dumbledore


controlplane ~ ➜  k describe netpol 
Name:         default-allow-all
Namespace:    default
Created on:   2025-01-01 13:37:41 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     i-am=dumbledore
  Allowing ingress traffic:
    To Port: <any> (traffic allowed to all ports)
    From: <any> (traffic not restricted by source)
  Allowing egress traffic:
    To Port: <any> (traffic allowed to all ports)
    To: <any> (traffic not restricted by destination)
  Policy Types: Egress, Ingress


  ```
  </details>



<details>
<summary>
Senario -
Task 3: Set Policies for "potions" Namespace
In the potions namespace:

Create a network policy, potions-rules, to restrict ingress and egress traffic to pods with specific labels. (Hint: Use pod selectors in both ingress and egress sections)
Policy potions-rules limits ingress and egress traffic to pods with labels class: potions

Create another policy, potions-port-rule, to allow egress traffic to a specific namespace but only to a specific port. (Hint: Use a namespaceSelector and specify the port in the egress rule)
Policy potions-port-rule allows egress traffic to the darkarts namespace but only on port 80

</summary>

```bash
controlplane ~ ➜  cat potions-rules.yaml 

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name : potions-rules
 namespace: potions 
spec:
  policyTypes:
    - Egress
    - Ingress
  ingress:
    - from:
       - podSelector:
            matchLabels:
              class: potions
  egress:
    - to:
       - podSelector:
           matchLabels:
              class: potions
  podSelector: {}


controlplane ~ ➜  k describe netpol -n potions
Name:         potions-rules
Namespace:    potions
Created on:   2025-01-01 14:24:09 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Allowing ingress traffic:
    To Port: <any> (traffic allowed to all ports)
    From:
      PodSelector: class=potions
  Allowing egress traffic:
    To Port: <any> (traffic allowed to all ports)
    To:
      PodSelector: class=potions
  Policy Types: Egress, Ingress


controlplane ~ ➜  vi potions-port-rule.yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name : potions-port-rule
 namespace: potions
spec:
  policyTypes:
    - Egress
  egress:
    - to:
       - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: darkarts
      ports:
           -  port: 80
              protocol: TCP
           -  port: 80
              protocol: UDP
  podSelector: {} 

controlplane ~ ➜  k describe netpol -n potions
Name:         potions-port-rule
Namespace:    potions
Created on:   2025-01-01 14:38:30 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Not affecting ingress traffic
  Allowing egress traffic:
    To Port: 80/TCP
    To Port: 80/UDP
    To:
      NamespaceSelector: kubernetes.io/metadata.name=darkarts
  Policy Types: Egress

  ```
  </details>



<details>
<summary>
Senario -
Task 4: Secure "darkarts" Namespace
Lastly, in the darkarts namespace:

Create a network policy, darkarts-magic, to deny all ingress traffic except from a specific pod in a specific namespace. (Hint: Use a combination of podSelector and namespaceSelector in your ingress rule)
The darkarts-magic policy should also allow egress traffic on a specific port and CIDR range. (Hint: Use ports and ipBlock under the egress rule)
Policy darkarts-magic denies all ingress traffic except from dumbledore pod in the default namespace (using existing labels)
Policy darkarts-magic only allows egress traffic to port 53 (both TCP and UDP) to IPs in CIDR range of 10.0.0.0/24

Create a darkarts-no-access network policy to restrict all ingress and egress traffic. (Hint: You don't need to specify any from or to rules, just the policyTypes)
Policy darkarts-no-access blocks all ingress traffic
Policy darkarts-no-access blocks all egress traffic

</summary>

```bash

controlplane ~ ➜  cat darkarts-magic.yaml 


apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name : darkarts-magic
 namespace: darkarts
spec:
  policyTypes:
    - Egress
    - Ingress
  ingress:
    - from:
      - podSelector: 
           matchLabels:
             i-am: dumbledore
        namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: default
  egress:
    - to:
      - ipBlock:
          cidr: 10.0.0.0/24
      ports:
           -  port: 53
              protocol: TCP
           -  port: 53
              protocol: UDP
  podSelector: {}


  
controlplane ~ ➜  k describe  netpol -n darkarts
Name:         darkarts-magic
Namespace:    darkarts
Created on:   2025-01-01 14:51:47 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Allowing ingress traffic:
    To Port: <any> (traffic allowed to all ports)
    From:
      NamespaceSelector: kubernetes.io/metadata.name=default
      PodSelector: i-am=dumbledore
  Allowing egress traffic:
    To Port: 53/TCP
    To Port: 53/UDP
    To:
      IPBlock:
        CIDR: 10.0.0.0/24
        Except: 
  Policy Types: Egress, Ingress



controlplane ~ ➜  cat darkarts-no-access.yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name : darkarts-no-access
  namespace: darkarts
spec:
  policyTypes:
    - Egress
    - Ingress
  ingress: []
  egress: []
  podSelector: {}


controlplane ~ ✖ k describe netpol -n darkarts

Name:         darkarts-no-access
Namespace:    darkarts
Created on:   2025-01-01 15:02:16 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Allowing ingress traffic:
    <none> (Selected pods are isolated for ingress connectivity)
  Allowing egress traffic:
    <none> (Selected pods are isolated for egress connectivity)
  Policy Types: Egress, Ingress

  ```
  </details>

