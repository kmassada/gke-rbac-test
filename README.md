# GKE RBAC

Set global variables

```
namepsace=what
role=viewer
app=dat
```

create a namespace to test on

```
kubectl create namespace $namespace
```

create a service account

```
kubectl create serviceaccount $namespace-$app-sa -n $namespace
```

bind that service account to the cluster role of viewer

```
kubectl create rolebinding $namespace-$app-rolebinding \
  --clusterrole=$role  \
  --serviceaccount=$namespace:$namespace-$app-sa \
  --namespace=$namespace
```

list the resources you just created

```
kubectl get serviceaccount,rolebinding -n $namespace
```

## test permissions at shell

you can certify this: 

```
$ kubectl delete pod $anotherpod --as system:serviceaccount:$namespace:$namespace-$app-sa
```

## test permissions in a pod

create nginx deployment and override service account

```
kubectl run nginx --image=nginx --overrides='{ "spec": { "template": { "spec": { "serviceAccount": "'"$namespace-$app-sa"'" } } } }' -n $namespace
```

scale the deployment to 3 pods

```
kubectl scale --replicas=3 deployment/nginx -n $namespace
```

list all the pods

```
kubectl get pods -n $namespace
```

pick one of the nginx pods and shell into it

```
kubectl exec -it $podname -n $namespace -- bash
```

install system tools

```
root@$podname:/# curl
bash: curl: command not found
root@$podname:/# apt-get update && apt-get install curl -y
```

try to delte pods or other resources in `-n default` to observe behavior

```
root@$podname:/# curl -ik -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" https://kubernetes/api/v1/namespaces/default/pods 

root@$podname:/# curl -ik -X "DELETE" -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" https://kubernetes/api/v1/namespaces/default/pods/$podname
```

## Debug

here are a few troubleshooting steps

### Verify DELETE

validate the successful `DELETE` call by checking the GKE console logs

```
resource.type="k8s_cluster"
protoPayload.methodName="io.k8s.core.v1.pods.delete"
```

### Verify correct token is used

validate the right token is mounted

```
volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
name: $namespace-$app-sa
```

### legacy ABAC

unusual behavior?

Verify legacyAbac

```
gcloud container clusters describe $clustername --zone=$zone --format json | jq .legacyAbac
{}
```

if true alter it

```
gcloud container clusters update $clustername --zone=$zone --no-enable-legacy-authorizatio
```