---
title: "File Access Monitoring"
weight: 4
description: "File access traces with Tetragon"
---

Tracing policies can be added to Tetragon through YAML configuration files
that extend Tetragon's base execution tracing capabilities. These policies
perform filtering in kernel to ensure only interesting events are published
to userspace from the BPF programs running in kernel. This ensures overhead
remains low even on busy systems.

The instructions below extend the example from [Execution Monitoring](/tetragon/docs/getting-started/d/execution)
with a policy to monitor sensitive files in Linux. The policy used is
[`file_monitoring.yaml`](https://github.com/cilium/tetragon/blob/main/examples/quickstart/file_monitoring.yaml),
which you can review and extend as needed. Files monitored here serve as a good
base set of files.

## Apply the tracing policy

To apply the policy in Kubernetes, use `kubectl`. In Kubernetes, the policy
references a Custom Resource Definition (CRD) installed by Tetragon. Docker uses
the same YAML configuration file as Kubernetes, but this file is loaded from
disk when the Docker container is launched.

Note that these instructions assume you've installed the demo application, as
outlined in either the [Quick Kubernetes Install](/tetragon/docs/getting-started/d/install-k8s)
or the [Quick Docker Install](/tetragon/docs/getting-started/d/install-docker)
section.
kubectl apply -f https://raw.githubusercontent.com/cilium/tetragon/main/examples/quickstart/file_monitoring.yaml
kubectl apply -f https://raw.githubusercontent.com/cilium/tetragon/main/examples/quickstart/file_monitoring.yaml
wget https://raw.githubusercontent.com/cilium/tetragon/main/examples/quickstart/file_monitoring.yaml
docker stop tetragon
docker run -d --name tetragon --rm --pull always \
  --pid=host --cgroupns=host --privileged \
  -v ${PWD}/file_monitoring.yaml:/etc/tetragon/tetragon.tp.d/file_monitoring.yaml \
  -v /sys/kernel/btf/vmlinux:/var/lib/tetragon/btf \
  quay.io/cilium/tetragon:latest
## Observe Tetragon file access events

With the tracing policy applied you can attach `tetra` to observe events again:
kubectl exec -ti -n kube-system ds/tetragon -c tetragon -- tetra getevents -o compact --pods xwing
POD=$(kubectl -n kube-system get pods -l 'app.kubernetes.io/name=tetragon' -o name --field-selector spec.nodeName=$(kubectl get pod xwing -o jsonpath='{.spec.nodeName}'))
kubectl exec -ti -n kube-system $POD -c tetragon -- tetra getevents -o compact --pods xwing
docker exec -ti tetragon tetra getevents -o compact
To generate an event, try to read a sensitive file referenced in the policy.
kubectl exec -ti xwing -- bash -c 'cat /etc/shadow'
kubectl exec -ti xwing -- bash -c 'cat /etc/shadow'
cat /etc/shadow
This will generate a read event (Docker events will omit Kubernetes metadata
shown below) that looks something like this:

```
ðŸš€ process default/xwing /bin/bash -c "cat /etc/shadow"
ðŸš€ process default/xwing /bin/cat /etc/shadow
ðŸ“š read    default/xwing /bin/cat /etc/shadow
ðŸ’¥ exit    default/xwing /bin/cat /etc/shadow 0
```

Per the tracing policy, Tetragon generates write events in responses to attempts
to write in sensitive directories (for example, attempting to write in the
`/etc` directory).
kubectl exec -ti xwing -- bash -c 'echo foo >> /etc/bar'
kubectl exec -ti xwing -- bash -c 'echo foo >> /etc/bar'
echo foo >> /etc/bar
In response, you will see output similar to the following (Docker events do not
include the Kubernetes metadata shown here).

```
ðŸš€ process default/xwing /bin/bash -c "echo foo >>  /etc/bar"
ðŸ“ write   default/xwing /bin/bash /etc/bar
ðŸ“ write   default/xwing /bin/bash /etc/bar
ðŸ’¥ exit    default/xwing /bin/bash -c "echo foo >>  /etc/bar
```

## What's next

To explore tracing policies for networking see the [Networking Monitoring](/tetragon/docs/getting-started/d/network)
section of the Getting Started guide.
To dive into the details of policies and events please see the [Concepts](https://github.com/cilium/tetragon/tree/main/docs/concepts)
section of the documentation.
