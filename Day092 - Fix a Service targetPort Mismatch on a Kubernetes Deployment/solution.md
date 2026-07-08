# Solution

A Kubernetes Service gives a set of pods a stable virtual IP and routes traffic to them, using `port` (what clients dial) and `targetPort` (the container port kube-proxy forwards to). This task hands you a green Deployment whose Service silently routes nowhere, and you fix the `targetPort` mismatch so the Service reaches its backing pods.

> As an MLOps engineer, you make the model server reachable by fixing its Kubernetes Service wiring — you are not changing the model; the served container is an `nginx:alpine` stand-in for the model server.

#### Follow the steps below

##### 1. Read the cluster state.
From a VS Code terminal:
```
kubectl get deploy fraud-detector
kubectl get svc fraud-detector-svc
kubectl get endpoints fraud-detector-svc -o wide
```

The Deployment is `Available 2/2`. The Service exists. The Endpoints row shows addresses populated at port `8080`.

##### 2. Compare container port vs. Service targetPort.
```
kubectl get deploy fraud-detector -o jsonpath='{.spec.template.spec.containers[0].ports[0].containerPort}'
kubectl get svc fraud-detector-svc -o jsonpath='{.spec.ports[0].targetPort}'
```

`containerPort` reads `80`. `targetPort` reads `8080`. Mismatch — kube-proxy is forwarding to a port the container isn't listening on.

##### 3. Fix the Service manifest.
Open `/root/code/k8s/service.yaml` in the VS Code editor. Change:
```yaml
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30092
```
to:
```yaml
  ports:
    - port: 8080
      targetPort: 80
      nodePort: 30092
```
Save.

##### 4. Re-apply.
```
kubectl apply -f /root/code/k8s/service.yaml
kubectl get endpoints fraud-detector-svc -o wide
```

Endpoints now list each pod IP with port `80`.

##### 5. End-to-end probe.
```
kubectl run probe --rm -i --restart=Never --image=busybox -- \
  wget -qO- http://fraud-detector-svc:8080/ | head -5
```

The nginx welcome page prints. The Service is now routing to the pods.

#### References

- Kubernetes — Service (`port` vs `targetPort`, selectors, Endpoints): https://kubernetes.io/docs/concepts/services-networking/service/
- Kubernetes — debugging Services (empty/mis-pointed Endpoints, in-cluster probing): https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/
