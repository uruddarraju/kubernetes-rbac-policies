Cluster RBAC Policies
========================
I will list down all the RBAC Policies needed for the functioning of a Kube cluster with only the RBAC Authorizer below on a component by component basis

**Default Role**
Given to all users in the system, would help in discovery and common read only operations

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: default-reader
rules:
  - apiGroups: [""]
    resources:
      - componentstatuses
      - events
      - endpoints
      - namespaces
      - nodes
      - persistentvolumes
      - resourcequotas
      - services
    verbs: ["get", "watch", "list"]
  - nonResourceURLs: ["*"]
    verbs: ["get", "watch", "list"]
```
Appropriate binding would be:
```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: default-reader-role-binding
subjects:
  - kind: User
    name: "*"
roleRef:
  kind: ClusterRole
  name: default-reader
  apiVersion: rbac.authorization.k8s.io/v1alpha1
```

**Read all**
Can be given to pseudo admins (like schedulers), for readonly operations. Not given by default to anyone. Can read everything except secrets
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1alpha1
kind: ClusterRole
metadata:
  name: cluster-read-all
rules:
  -
    apiGroups:
      - ""
      - apps
      - autoscaling
      - batch
      - extensions
      - policy
      - rbac.authorization.k8s.io
    resources:
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs:
      - get
      - watch
      - list
  - nonResourceURLs: ["*"]
    verbs:
      - get
      - watch
      - list

```

**Cluster Administrator**
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1alpha1
kind: ClusterRole
metadata:
  name: cluster-admin-all
rules:
  -
    apiGroups:
      - ""
      - apps
      - autoscaling
      - batch
      - extensions
      - policy
      - rbac.authorization.k8s.io
    resources:
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["*"]
  - nonResourceURLs: ["*"]
    verbs: ["*"]
```

**Controller Manager**
Controller manager needs access to almost all the resources in Kubernetes hence, we need to grant it top level admin access. Controller manager is actually a super user, so it can work even without a rolebinding.
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1alpha1
kind: ClusterRole
metadata:
  name: cluster-controller-manager
rules:
  -
    apiGroups:
      # have access to everything except Secrets
      - "*"
    resources: ["*"]
    verbs: ["*"]
  - nonResourceURLs: ["*"]
    verbs: ["*"]
```

**Kubelet**

To spin up pods and update node and pod status the kubelet would need the following role and binding:
```yaml
apiVersion: rbac.authorization.k8s.io/v1alpha1
kind: ClusterRole
metadata: 
  name: kubelet-runtime
rules:
- apiGroups:
  - ""
  attributeRestrictions: null
  resources:
  - configmaps
  - persistentvolumes
  - persistentvolumeclaims
  - secrets
  - services
  - healthz
  verbs:
  - get
  - watch
  - list
- attributeRestrictions: null
  nonResourceURLs:
  - '*'
  verbs:
  - get
  - watch
  - list
- apiGroups:
  - ""
  attributeRestrictions: null
  resources:
  - events
  - nodes
  - nodes/status
  - pods
  - pods/status
  verbs:
  - '*'
- attributeRestrictions: null
  nonResourceURLs:
  - '*'
  verbs:
  - '*'
```

The appropriate binding would be:
```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: kubelet-role-binding
subjects:
  - kind: User
    name: kubelet
roleRef:
  kind: ClusterRole
  name: kubelet-runtime
  apiVersion: rbac.authorization.k8s.io/v1alpha1
```

* For kubelet to check apiserver `healthz`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1alpha1
kind: ClusterRole
metadata:
  name: healthz-reader-role
rules:
  -
    apiGroups:
      - ""
    resources: []
    verbs: ["get"]
  - nonResourceURLs: ["*"]
    verbs: ["get"]
```
The reason we made this explicit healthz binding is to make sure only the healthz is allowed access by everyone. Not only kubelet, but other monitoring systems can usee healthz in the future and hence granting access to all users
```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: healthz-role-binding
subjects:
  - kind: User
    name: "*"
roleRef:
  kind: ClusterRole
  name: healthz-reader-role
  apiVersion: rbac.authorization.k8s.io/v1alpha1
```

**Scheduler**
Scheduler needs the following roles:

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: scheduler
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["*"]
  - nonResourceURLs: ["*"]
    verbs: ["*"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["update"]
    nonResourceURLs: [""]
    verbs: ["*"]
```

needs the following role bindings:
```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: scheduler-role-binding
subjects:
  - kind: User
    name: system:scheduler
roleRef:
  kind: ClusterRole
  name: cluster-read-all
  apiVersion: rbac.authorization.k8s.io/v1alpha1
```
and also
```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: scheduler-role-binding
subjects:
  - kind: User
    name: system:scheduler
roleRef:
  kind: ClusterRole
  name: scheduler
  apiVersion: rbac.authorization.k8s.io/v1alpha1
```

**Kube Proxy** 
Needs the following:
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1alpha1
kind: ClusterRole
metadata:
  name: kube-proxy-role
rules:
  -
    apiGroups:
      - ""
    resources:
      - endpoints
      - events
      - services
      - nodes
    verbs: ["get", "watch", "list"]
  - nonResourceURLs: ["*"]
    verbs: ["get", "watch", "list"]
  -
    apiGroups:
      - ""
    resources:
      - events
    verbs: ["*"]
  - nonResourceURLs: ["*"]
    verbs: ["*"]
```

```yaml
# This cluster role binding allows anyone in the "manager" group to read secrets in any namespace.
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: kubeproxy-role-binding
subjects:
  - kind: User
    name: kube_proxy
roleRef:
  kind: ClusterRole
  name: kube-proxy-role
  apiVersion: rbac.authorization.k8s.io/v1alpha1
```


**Kube Proxy**
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1alpha1
kind: ClusterRole
metadata:
  name: kube-proxy-role
rules:
  -
    apiGroups:
      - ""
    resources:
      - endpoints
      - events
      - services
      - nodes
    verbs: ["get", "watch", "list"]
  - nonResourceURLs: ["*"]
    verbs: ["get", "watch", "list"]
  -
    apiGroups:
      - ""
    resources:
      - events
    verbs: ["*"]
  - nonResourceURLs: ["*"]
    verbs: ["*"]

```
And the corresponding binding

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: kube-system-sa-admin
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: default
    namespace: kube-system
roleRef:
  kind: Role
  namespace: kube-system
  name: kube-system-admin
  apiVersion: rbac.authorization.k8s.io/v1alpha1
```

**Kube System Components:**
Long term plan should be to move components out of kube-system into appropriate namespaces. DNS, Monitoring etc.

The following is a `Role` and grants access only within the kube-system namespace
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  namespace: kube-system
  name: kube-system-admin
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
```

Appropriate binding:
```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: kube-system-sa-admin
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: default
    namespace: kube-system
roleRef:
  kind: Role
  namespace: kube-system
  name: kube-system-admin
  apiVersion: rbac.authorization.k8s.io/v1alpha1
```
  
Read all for dns and monitoring to work
```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: kube-system-readall-role-binding
subjects:
  - kind: ServiceAccount
    name: default
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-read-all
  apiVersion: rbac.authorization.k8s.io/v1alpha1
```
