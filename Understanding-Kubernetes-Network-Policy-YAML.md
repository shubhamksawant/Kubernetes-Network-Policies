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

