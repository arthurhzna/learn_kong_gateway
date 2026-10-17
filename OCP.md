# Docker Commands Equivalent in OpenShift

This guide explains how to inspect applications and resources in OpenShift using the `oc` CLI as an alternative to Docker CLI commands.

## 1. Pods vs. Deployments

In Docker, `docker ps` lists running containers, while `docker ps -a` lists all containers, including stopped ones.

OpenShift works differently:

- **`oc get pods`** lists Pod resources in the active namespace. Completed or failed Pods may still appear until their objects are deleted.
- **`oc get deploy`** lists all Deployments, including applications with zero replicas.

> **Current state:** All applications have been scaled to `0` according to our agreement. Therefore, application Pods are not running, and only the terminal Pod or other independently running Pods may remain.

## 2. Docker vs. OpenShift Commands

| Docker CLI | OpenShift CLI (`oc`) | Description |
|---|---|---|
| `docker ps` | `oc get pods` | List existing Pods |
| `docker ps -a` | `oc get deploy` | List Deployments, including those with zero replicas |
| `docker logs <container>` | `oc logs <pod>` | View Pod logs |
| `docker inspect <container>` | `oc describe pod <pod>` | Inspect Pod details and status |
| `docker inspect <container>` | `oc get pod <pod> -o yaml` | View Pod configuration in YAML |
| `docker exec -it <container> sh` | `oc rsh <pod>` | Open a shell inside a container in a Pod |
| `docker images` | `oc get is` | List ImageStreams, if used |
| `docker network ls` | `oc get svc` | Inspect Services used for internal networking |
| Port mapping | `oc get svc` and `oc get routes` | Inspect Service ports and external access |
| `docker start/stop` | `oc scale --replicas=<n> deployment/<name>` | Change the desired number of Deployment replicas |

**Important notes:**
- `oc get pods` is not an exact equivalent of `docker ps`. Completed or failed Pods may still be listed.
- `oc get deploy` is not an exact equivalent of `docker ps -a`. A Deployment manages application replicas; it is not a list of all containers.
- `oc rsh` may provide access to containers owned by other teams. Use it only when authorized.
- **Do not run `oc scale` to start or stop applications.** Keep replica counts unchanged according to our agreement.

## 3. Application Status Summary

The following script generates a summary of Deployments, replica status, container ports, Service ports, and internal Kubernetes DNS names.

```bash
NS=$(oc project -q)

oc get svc -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.ports[*]}{.port}{" "}{end}{"\n"}{end}' > /tmp/svc.txt

oc get deploy -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.replicas}{"\t"}{.status.readyReplicas}{"\t"}{.spec.template.spec.containers[0].ports[*].containerPort}{"\n"}{end}' > /tmp/dep.txt

awk -F'\t' -v ns="$NS" '
BEGIN {
    OFS="\t"
    print "NAME","STATUS","CONTAINER_PORT","SVC_PORT","DNS_INTERNAL"
}
NR==FNR {
    svc[$1]=$2
    next
}
{
    desired=($2=="" ? 0 : $2)
    ready=($3=="" ? 0 : $3)

    if (desired == 0)
        status="Stopped"
    else if (ready == desired)
        status="Up"
    else
        status="NotReady"

    status=status "(" ready "/" desired ")"

    if ($1 in svc) {
        sp=svc[$1]
        sub(/ +$/, "", sp)
        dns=$1 "." ns ".svc.cluster.local"
    } else {
        sp="-"
        dns="-"
    }

    print $1, status, ($4=="" ? "-" : $4), sp, dns
}' /tmp/svc.txt /tmp/dep.txt | column -t -s $'\t'
```

### Example output

```text
NAME          STATUS         CONTAINER_PORT  SVC_PORT  DNS_INTERNAL
api-service   Stopped(0/0)   8080            8080      api-service.project.svc.cluster.local
worker        Stopped(0/0)   9000            -         -
```

*The output above is illustrative. Actual results depend on the resources in the active namespace.*

### Column definitions

| Column | Description |
|---|---|
| `NAME` | Deployment name |
| `STATUS` | Summary based on desired and ready replica counts |
| `CONTAINER_PORT` | Container port from the first container in the Deployment template |
| `SVC_PORT` | Service port, if a Service with the same name as the Deployment exists |
| `DNS_INTERNAL` | Internal Service DNS name, if a matching Service exists |

Status formats:

- `Stopped(0/0)`: The Deployment is configured with zero desired replicas.
- `Up(1/1)`: All desired replicas are Ready.
- `NotReady(0/1)`: One replica is desired, but none is Ready.

**Limitations:** This script provides a practical summary, not a complete application health check. It assumes the Service name matches the Deployment name and reads container ports only from the first container. Deployments with multiple containers or differently named Services require additional inspection.

## 4. Additional Inspection Commands

Use these read-only commands for troubleshooting:

```bash
# Check the active namespace
oc project -q

# List all Deployments
oc get deploy

# List existing Pods
oc get pods

# List Services and ports
oc get svc

# List Routes and external hostnames
oc get routes

# Inspect Deployment details
oc describe deploy <deployment-name>

# View Deployment configuration
oc get deploy <deployment-name> -o yaml

# View Pod logs, if the Pod exists
oc logs <pod-name>
```

## 5. Usage Guidelines

1. Use read-only commands for inspection and troubleshooting.
2. Do not change replica counts or restart applications without authorization.
3. Do not access containers belonging to other teams without permission.
4. Internal DNS names are usable only by workloads with appropriate cluster connectivity and configuration.
5. When a Deployment has zero replicas, no application container is running to accept connections or generate new application logs.

**Purpose:** Provide a safe and consistent way to inspect application resources in OpenShift without changing the agreed Deployment state.
