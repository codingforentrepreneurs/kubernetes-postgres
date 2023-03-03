# PostgreSQL on Kubernetes

This repo exists as a public reference for [this course](https://www.codingforentrepreneurs.com/courses/terraforming-kubernetes-github-actions/). If you follow that course, you will know exactly how to execute the steps below as well as what they all mean.


## Getting Startted

First, we need the following (in this order)

1. A kubernetes cluster
2. The `apps` namespace
3. A service account with correct Role and RoleBinding for the `apps` namespace via [this blog post](https://www.codingforentrepreneurs.com/blog/kubernetes-rbac-service-account-github-actions/)


## Example Admin Project

```
mkdir -p ~/dev/tf-k8s-admin
cd ~/dev/tf-k8s-admin
git clone https://github.com/codingforentrepreneurs/terraforming-kubernetes-rapid .
```

Create an account on [Akamai Linode](https://www.linode.com/cfe) and get an API Key in your linode account [here](https://www.linode.com/cfe).

```
echo "linode_api_token=\"YOUR_API_KEY\"" >> terraform.tfvars
```

Apply changes:

```
terraform apply
```

Create the service account:

```
kubectl create namespace apps
kubectl create serviceaccount github-actions-sa -n apps
```

A shortcut directly from [the blog post](https://www.codingforentrepreneurs.com/blog/kubernetes-rbac-service-account-github-actions/), create a role/rolebinding manifest (e.g `~/dev/tf-k8s-admin/service-accounts/role-binding.yaml`) with
```yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: github-actions-role
  namespace: apps
rules:
  - apiGroups: [""]
    resources:
      - configmaps
      - persistentvolumeclaims
      - pods
      - pods/exec
      - pods/log
      - pods/portforward
      - secrets
      - services
    verbs:
      - get
      - watch
      - list
      - update
      - create
      - patch
      - delete
  - apiGroups: ["apps"]
    resources:
      - deployments
      - statefulsets
    verbs:
      - get
      - list
      - watch
      - update
      - create
      - patch
      - delete
--- 
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: github-actions-rolebinding
  namespace: apps
subjects:
  - kind: ServiceAccount
    name: github-actions-sa
    namespace: default
roleRef:
  kind: Role
  name: github-actions-role
  apiGroup: rbac.authorization.k8s.io
```

Apply the changes:

```
kubectl apply -f ~/dev/tf-k8s-admin/service-accounts/role-binding.yaml
```


Get the service account manifest (again from [this blog post](https://www.codingforentrepreneurs.com/blog/kubernetes-rbac-service-account-github-actions/) ):


```bash
export SERVICE_ACCOUNT="github-actions-sa"
export SERVICE_ACCOUNT_NAMESPACE="apps"
export SERVICE_ACCOUNT_SECRET_NAME=$(kubectl get serviceaccounts $SERVICE_ACCOUNT -n $SERVICE_ACCOUNT_NAMESPACE -o jsonpath="{.secrets[0].name}")

export SERVICE_ACCOUNT_SECRET_CERT=$(kubectl get secret $SERVICE_ACCOUNT_SECRET_NAME -n $SERVICE_ACCOUNT_NAMESPACE -o jsonpath="{.data['ca\.crt']}")
export SERVICE_ACCOUNT_SECRET_TOKEN=$(kubectl get secret $SERVICE_ACCOUNT_SECRET_NAME -n $SERVICE_ACCOUNT_NAMESPACE  -o jsonpath="{.data.token}" | base64 -d)
export CLUSTER_URL=$(kubectl config view --minify -o 'jsonpath={.clusters[0].cluster.server}')
export CLUSTER_NAME=$(kubectl config view --minify -o 'jsonpath={.clusters[0].name}')
```

Create the `kubeconfig-sa.yaml` file (be sure to add `kubecfongi-sa.yaml` to `.gitignore`):


```bash
cat << EOF > kubeconfig-sa.yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: $SERVICE_ACCOUNT_SECRET_CERT
    server: $CLUSTER_URL
  name: $CLUSTER_NAME
contexts:
- context:
    cluster: $CLUSTER_NAME
    user: $SERVICE_ACCOUNT
  name: $CLUSTER_NAME-ctx
current-context: $CLUSTER_NAME-ctx
kind: Config
users:
- name: $SERVICE_ACCOUNT
  user:
    token: $SERVICE_ACCOUNT_SECRET_TOKEN
EOF

```

In your forked version of [this repo](https://github.com/codingforentrepreneurs/kubernertes-postgres), update your Github Actions Secrets with:

`KUBECONFIG_SA` with the contents of the newly created `kubeconfig-sa.yaml` 


Run the workflow `.github/workflows/start-pg.yaml` and celebrate!

Consider using: 

```
kubectl port-forward statefulset/postgres-statefulset 5422:5432
```

Then:

```
PGDATABASE=cfedb psql -U postgres -h localhost -p 5422
```

