### Understanding the Kubernetes Network Policy YAML
A Network Policy in Kubernetes is a robust security feature that permits you to manage network communication to and from pods in a cluster. Just like any other object in Kubernetes, it is defined using a YAML file.

This guide delves into the structure of a Kubernetes Network Policy YAML file, with a focus on the common fields under the spec section.

YAML File Structure
In the previous lab, we used a sample deny-all network policy without thoroughly understanding its structure. Now, let's decipher it using a more straightforward example:

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: simple-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
  ```

Let us now look at each of the fields.

apiVersion
This field indicates the version of the API used to create this Network Policy. For Network Policies, this is usually networking.k8s.io/v1.

kind
This field signifies the type of Kubernetes resource being defined. For Network Policies, this will always be NetworkPolicy.

metadata
This section encompasses the metadata related to the Network Policy, including the name of the policy and the namespace in which the policy is to be implemented. Network Policies are namespace-scoped, which means they apply to a specific namespace and affect only the pods within that namespace.

spec
The spec is the actual specification of the Network Policy. It contains the following fields:

podSelector
This field determines which pods the policy applies to. The selector uses label selection to target pods. In the example above, the policy is applied to pods labeled role: db.

If podSelector isn't specified or is empty ({}), it selects all pods in the namespace.

An empty set of curly braces {} is used in Kubernetes Network Policy to signify everything or all within the context of the field it's used in.

For example, in a podSelector field, {} means all pods. This means that the network policy applies to all pods within the namespace.

```bash
podSelector: {}
  ```

policyTypes
This field defines whether the policy applies to Ingress, Egress, or both. In the example above, the policy applies to Ingress traffic. We'll cover Egress in a later lab.

ingress
The ingress field defines incoming traffic rules for the targeted pods. In the example, the ingress rule allows traffic from pods labeled with role: frontend.

Each ingress rule is made up of from and ports fields. The from field accepts a list of sources from which traffic is allowed. Each item in the list can be a podSelector (as in the example), namespaceSelector, or ipBlock. We will learn more about these in the upcoming labs.

An empty set of square brackets [] in Kubernetes Network Policy is used to denote none or no traffic.

For example, in the ingress field, [] signifies that no traffic is allowed into the pods that the policy applies to.

```bash
ingress: []
  ```
For simplicity, we are only focusing on ingress rules in this lab. We will look ategress rules, which deal with outbound traffic, in a subsequent lab.

Now putting all of this together, the Sample YAML does the following:

metadata: - This section provides metadata about the Network Policy.

name: simple-network-policy - This is the name of the Network Policy.

namespace: default - This indicates the namespace where the policy will be applied. Network Policies are namespace-scoped, meaning they only affect pods within the defined namespace.

spec: - This section contains the actual specification of the Network Policy. It includes the following fields:

podSelector: - This field determines which pods the policy applies to. Here, it's selecting all pods in the namespace with a label of role: db. In other words, this policy will apply to "database" pods that have the label role: db.

policyTypes: - This field defines the type of traffic the policy will apply to. Here, the policy applies to Ingress traffic, which is incoming traffic to the pods.

ingress: - This field defines the rules for the incoming traffic to the pods that the policy applies to. In this case, the policy allows traffic from pods labeled with role: frontend.

The from section under ingress lists the sources that the incoming traffic can originate from. Each item in this list can either be a podSelector, namespaceSelector, or ipBlock. In this case, a podSelector is used to allow traffic from all pods labeled role: frontend.

In summary, this Network Policy allows incoming traffic to all pods labeled role: db in the default namespace, but only if the traffic is coming from pods labeled role: frontend. This would be a common setup for a web application, where frontend pods (e.g., a web server) need to communicate with backend database pods, but access is restricted for all other pods or external sources.


<details>
<summary>

Senario -
Which pods does the Network Policy in the following YAML file apply to?


```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: unique-network-policy-2
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ```
</summary>

pods with role:db  labels
</details>


<details>
<summary>
Senario -
What type of policy is implemented in the following Network Policy YAML file?

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: unique-network-policy-2
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ```
</summary>
Ingress

</details>


<details>
<summary>

Senario -
What does the ingress rule in the following Network Policy YAML file apply to?

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: unique-network-policy-3
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          project: frontend
  ```
</summary>

pods with ns   project: frontend
</details>


<details>
<summary>
Senario -
What does {} signify in the podSelector field in the following YAML file?

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: unique-network-policy-5
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
  ```
</summary>
all pods defauls ns

</details>

<details>
<summary>

Senario -
What does the ingress rule in the following Network Policy YAML file apply to?

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: unique-network-policy-4
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16

  ```
</summary>
all ip within range 172.17.0.0/16

</details>


<details>
<summary>
Senario -
Analyze the network policy YAML provided below and pick the correct behavior that will be enforced when this policy is applied:

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: unique-network-policy-6
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress: []
  ```
</summary>
No traffic for pods with role: db labels

</details>



<details>
<summary>
Senario -
What does the policy in the following YAML file achieve?

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: unique-network-policy-7
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector: {}
  ```
</summary>
allow trrafic from all ns to 

</details>



<details>
<summary>
The Role of Default Deny Network Policies
By default, Kubernetes does not restrict the flow of network traffic between Pods. However, to establish a baseline security measure, we can use a 'Default Deny' network policy that denies all network traffic to and from Pods within the namespace unless specified otherwise.

Implementing a Default Deny policy ensures no unauthorized access occurs in any of the Pods, thereby enhancing the security of your Kubernetes cluster.

Here is a simple example of how a Default Deny policy can be implemented for the default namespace:
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
  ```

The podSelector with {} implies that the policy applies to all Pods within the default namespace. The absence of ingress rules indicates that all inbound network traffic is denied by default.

Using Default Deny as a Baseline for Fine-Grained Access Control
After implementing a Default Deny policy, we've effectively created a blacklist that denies all network traffic by default. This acts as a baseline security measure, safeguarding our Pods from any unnecessary or potentially harmful network communication.

This policy serves as a solid foundation upon which we can define fine-grained access controls.

However, this policy by itself, although secure, will most likely break the application as no pods cant connect to any other pod!

default-deny

To enable the necessary communication paths, such as allowing the middleware to connect to the MySQL database, or the front-end to connect to the middleware, we can create additional network policies. These allow specific types of traffic, effectively creating a whitelist atop our baseline blacklist.

In our upcoming labs, we will look into how we can further extend our network policy to enable these communication paths, thus demonstrating the powerful and flexible nature of Kubernetes Network Policies.

Conclusion
Implementing a Default Deny policy in Kubernetes is a crucial step towards securing your application at the network level. It forms a baseline security measure, acting as a blacklist to deny all communications by default. This baseline can then be extended with additional policies to establish fine-grained control over the network traffic within your cluster, providing robust and flexible security for your multi-tier application.


Senario -
Create a new network policy called default-deny-ingress with the following requirements:


The policy should be applied to all pods in the default namespace (including frontend, middleware and mysql)
It should block all inbound traffic for all pods in the default namespace.


</summary>

Use the following YAML. The podSelector is {} which means all pods in the default namespace will be affected by the policy.

The ingress field is null in this case, which means that once applied this policy will block ingress for all pods in the default namespace.

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
  ```
</details>


<details>
<summary>
Senario -
default-deny-ingress network policy in the default namespace, all inbound connections for all pods will be restricted.

To do this, create a new network policy called default-allow-ingress in the default namespace, which, as the name suggests, would allow all ingress for all pods in the namespace.


Note: Do not delete any of the existing resources including the default-deny-ingress network policy. If this policy was not created as part of the previous task, it will be automatically deployed.


</summary>

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-allow-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
    - {}


controlplane ~ ➜  kubectl exec middleware -- nc -zv mysql-svc 3306
mysql-svc (10.111.228.81:3306) open
  ```

</details>


<details>
<summary>
Senario -
we now have two network policies in the default namespace of this kubernetes cluster:
A default-deny-ingress which blocks all ingress and,
A default-allow-ingress that allows all ingress communication.

will pod able to connect to other ?

</summary>

And we saw that while both policies are applied simultaneously, the allow rule is enfoced:
```bash
controlplane ~ ➜  kubectl exec middleware -- nc -zv mysql-svc 3306
mysql-svc (10.101.141.65:3306) open

controlplane ~ ➜  kubectl exec middleware -- nc -zv frontend-svc 80
frontend-svc (10.100.63.217:80) open
```

This is because Network Policies in Kubernetes are additive, meaning that if there is any Network Policy that allows a certain type of traffic, that traffic will be allowed even if another Network Policy would block it. This design choice is based on the principle of explicitly allowed over implicit deny.

you have one policy that denies all ingress traffic and one that allows all ingress traffic. Kubernetes will sum these policies, and the result is that all ingress traffic is allowed. The policy to allow traffic is considered an explicit rule that should be followed, even if there's a more general policy that would deny the traffic.

However, it's important to note that this does NOT mean that allow policies always take precedence over deny policies. Rather, all policies are evaluated, and if there is any policy that would allow the traffic, then the traffic is allowed.

This is why when designing your Network Policies, we typically start with a broad deny policy, and then add specific allow policies for just the traffic you want to permit. We will see more examples of this in the upcoming labs.

</details>




<details>
<summary>
Senario -
Update this file to define the ingress field while using [] to denote none or no traffic.
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
  ```

</summary>

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress-null
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress: []
  ```

</details>


