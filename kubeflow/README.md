# Kubeflow User Setup

This directory contains YAML files for setting up a new user on a Kubeflow cluster. This includes creating a user-specific profile (namespace), assigning administrative rights to that user within their namespace, and authorizing their access through the Istio ingress gateway.

## 1. Create a User Profile (`my-profile.yaml`)

The `my-profile.yaml` file defines a Kubeflow `Profile`. In Kubeflow, a Profile corresponds to a Kubernetes namespace that is dedicated to a user or a team.

```yaml
apiVersion: kubeflow.org/v1
kind: Profile
metadata:
  ## the profile name will be the namespace name
  ## WARNING: unexpected behavior may occur if the namespace already exists
  name: all4dich
spec:
  ## the owner of the profile
  ## NOTE: you may wish to make a global super-admin the owner of all profiles
  ##       and only give end-users view or modify access to profiles to prevent
  ##       them from adding/removing contributors
  owner:
    kind: User
    name: all4dich@gmail.com

  ## plugins extend the functionality of the profile
  ## https://github.com/kubeflow/kubeflow/tree/master/components/profile-controller#plugins
  plugins: []

  ## optionally create a ResourceQuota for the profile
  ## https://github.com/kubeflow/kubeflow/tree/master/components/profile-controller#resourcequotaspec
  ## https://kubernetes.io/docs/reference/kubernetes-api/policy-resources/resource-quota-v1/#ResourceQuotaSpec
  resourceQuotaSpec: {}
```

**Key aspects of this file:**

*   **`metadata.name: all4dich`**: This will create a namespace named `all4dich`. This namespace will house all the resources for this user.
*   **`spec.owner.name: all4dich@gmail.com`**: This designates `all4dich@gmail.com` as the owner of this profile. The owner typically has full control over the profile.

Applying this file will create the `all4dich` namespace and set `all4dich@gmail.com` as its owner.

## 2. Assign Contributor Role (`my-contributor-role.yaml`)

The `my-contributor-role.yaml` file creates a `RoleBinding` in Kubernetes. This RoleBinding grants the specified user (`all4dich@gmail.com`) administrative privileges within the namespace created by `my-profile.yaml` (i.e., the `all4dich` namespace).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: user-all4dich-gmail-com-clusterrole-admin
  namespace: all4dich
  annotations:
    role: ClusterRole/kubeflow-admin
    user: all4dich@gmail.com
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubeflow-admin
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: all4dich@gmail.com
```

**Key aspects of this file:**

*   **`metadata.namespace: all4dich`**: This specifies that the RoleBinding applies to the `all4dich` namespace.
*   **`roleRef.name: kubeflow-admin`**: This references the `kubeflow-admin` `ClusterRole`. This role typically includes permissions to manage all resources within a namespace.
*   **`subjects.name: all4dich@gmail.com`**: This assigns the `kubeflow-admin` role to the user `all4dich@gmail.com`.

Applying this file ensures that the user `all4dich@gmail.com` has the necessary permissions to work within their Kubeflow profile/namespace.

## 3. Authorize User Access (`my-contributor-auth.yaml`)

The `my-contributor-auth.yaml` file defines an Istio `AuthorizationPolicy`. This policy is crucial for allowing the user to access their Kubeflow services (like Notebooks, Pipelines, etc.) through the central Istio ingress gateway.

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: user-all4dich-gmail-com-clusterrole-admin
  namespace: my-profile
  annotations:
    role: ClusterRole/kubeflow-admin
    user: all4dich@gmail.com
spec:
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
              - "cluster.local/ns/kubeflow/sa/ml-pipeline-ui"
      when:
        - key: request.headers[kubeflow-userid]
          values:
            - all4dich@gmail.com
```

**Key aspects of this file:**

*   **`metadata.namespace: my-profile`**: This policy is applied in a namespace that the Istio ingress gateway watches for such policies. **Important:** In this example, the namespace is `my-profile`. For this to correctly authorize access to the `all4dich` namespace (created by `my-profile.yaml`), this AuthorizationPolicy should typically be in the `all4dich` namespace, or the `selector` field should be used to target workloads in the `all4dich` namespace. If `my-profile` is a separate, specific namespace for such policies, ensure it's configured correctly in your Kubeflow Istio setup.
*   **`spec.rules.when.key: request.headers[kubeflow-userid]`**: This rule checks for a header named `kubeflow-userid`.
*   **`spec.rules.when.values: - all4dich@gmail.com`**: The policy allows requests if the `kubeflow-userid` header has the value `all4dich@gmail.com`. This header is typically injected by Kubeflow's authentication system.
*   **`spec.rules.from.source.principals`**: This specifies which upstream services (like the Istio ingress gateway and ML Pipeline UI) are allowed to make requests that are then evaluated by this policy.

Applying this file allows the Istio ingress gateway to route requests to services in the user's namespace, provided the `kubeflow-userid` header matches `all4dich@gmail.com`.

**Note on `metadata.namespace`**: The `namespace` in `my-contributor-auth.yaml` is set to `my-profile`. This should ideally match the namespace created by `my-profile.yaml` (which is `all4dich`). If these are intended to be different, ensure your Kubeflow and Istio configurations correctly handle this setup. Typically, an AuthorizationPolicy like this would be created in the target user's namespace (`all4dich` in this case) to protect resources within that namespace. Alternatively, a single AuthorizationPolicy in a central namespace (like `istio-system` or a dedicated Kubeflow system namespace) could use a `selector` to target specific workloads across user namespaces if that's the desired architecture.

## 4. How to Apply

To apply these configurations to your Kubeflow cluster, use the `kubectl apply -f <filename>` command. It's generally recommended to apply them in the following order:

1.  **Profile:** Create the namespace and define ownership.
    ```bash
    kubectl apply -f kubeflow/my-profile.yaml
    ```

2.  **RoleBinding:** Grant permissions to the user within the new namespace.
    ```bash
    kubectl apply -f kubeflow/my-contributor-role.yaml
    ```

3.  **AuthorizationPolicy:** Allow access through Istio.
    ```bash
    kubectl apply -f kubeflow/my-contributor-auth.yaml
    ```

Make sure your `kubectl` context is configured to point to the correct Kubernetes cluster where Kubeflow is installed. If you modified the `metadata.name` in `my-profile.yaml` or the `metadata.namespace` in the other files, ensure these commands and file contents reflect your changes.
