# KARAT
### Kubernetes Authorization Reporting and Alerting Tool

Kubernetes' [RBAC authorization model](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) is incredibly powerful and flexible. Granting permissions and priveleges to users, groups, and service accounts is as simple as running a `kubectl` command or launching a Helm template.

However, this ease of use and flexibilty means unintended permission grants and privilege escalations can creep into your Kubernetes cluster over time. Imagine this scenario: a cluster admin launches an application with a service account that has the ability to view pods across all namespaces in the cluster in the `default` namespace. Later, they delete that app, but not the service account. A month later, that same admin adds a non-privileged user to the `default` namespace so that they can launch their app. Now, that non-privileged user can use the service account to view pods throughout the cluster.

KARAT adresses this problem by surfacing Kubernetes RBAC information in a way that is clear and actionable.

KARAT can be used in two ways:
1. As a cli that reports on the state of your cluster's RBAC permissions
2. As an http server that reports when undesired permission grants have been created. This can be hooked into an alert system like [Prometheus' Alertmanager](https://prometheus.io/docs/alerting/alertmanager/) to ensure your operations team can react in realtime.

### Example Usage
Use the cli to see who has admin or escalated privileges in the cluster
```
$ karat report

The following users, groups, and service account have direct admin access to the cluster
NAME         KIND             GRANTED BY                      NAMESPACE
jsmith       user             clusterRoleBinding:c-12345
dev-group    group            clusterRoleBinding:c-45678
defatult     serviceAccount   clusterRoleBinding:c-98765      kube-system

The following users and groups have indirect admin access to the cluster
NAME         KIND             OBTAINED THROUGH (resource namespace:name)
jdoe         user             serviceAccount default:svcAccount1

The following users, groups, and service accounts have direct elevated (but not full admin) privileges in the cluster:
NAME               KIND             GRANTED BY
network-admins     group            clusterRoleBinding:c-45454
...
```

Use the cli to take a point-in-time snapshot report of your cluster's permissions and compare it to the cluster's state at a later date
```
$ karat report snapshot --output-file permissions-4-1-2019.json
Report saved to permissions.json

# At a later date, compare the current state of your cluster to a previous report snapshot

$ karat report diff --input-file permissions-4-1-2019.json

No new users, groups, or service accounts have been granted admin access to the cluster

The following new users, groups, and service accounts have been granted elevated privileges
NAME         KIND             GRANTED BY                    
bmurray      user             clusterRoleBinding:c-12121
```

Launch KARAT as an http server and check your cluster's state
```
$ karat launch-server --listen 0.0.0.0:8080 --config karat-config.yaml
karat listening 0.0.0.0:8080
# karat-config.yaml can outline your cluster's expected auth state and what you wish to alert on

# Status is ok because there are not unexpected privilege escalations
$ curl https://karat-server:8080/status
curl -k -i https://localhost:6443
HTTP/2 200
content-type: application/json

{
  "status": "ok"
}

# Status is "failed" because there are unexpected privilege escalations
$ curl https://karat-server:8080/status
curl -k -i https://localhost:6443
HTTP/2 418
content-type: application/json

{
  "status": "failed"
  // details on the unexpected privileges
}
```
Using curl is an oversimplication, but hopefully it illustrates how KARAT can be integrated with an alert manager to let you know when your cluster has been compromised.


