+++
title = 'Deploying an Application on Kubernetes'
description = 'A practical walkthrough for shipping a containerized web application with a Deployment and Service, then verifying the rollout safely.'
summary = 'Build a small but production-minded Kubernetes deployment: declare the workload, expose it through a Service, watch the rollout, and know where to look when something fails.'
date = '2026-08-02T09:00:00+01:00'
draft = false
authors = ['Mohamed Lembarki', 'Ihsane Elkhaldi']
topics = ['Kubernetes', 'Containers', 'DevOps']
+++

Kubernetes deployments become much easier to reason about when we separate the process into three questions: **what should run, how should Kubernetes keep it healthy, and how should traffic reach it?** In this walkthrough, we will deploy a small NGINX web application using a `Deployment` and a `Service`.

The same structure applies to most stateless applications. Replace the image, ports, health checks, and resource values with those required by your service.

## Before you begin

You need a running Kubernetes cluster and `kubectl` configured to communicate with it. A local cluster such as kind or minikube is enough for this example.

Confirm that the connection works:

```bash
kubectl cluster-info
kubectl get nodes
```

Create a dedicated namespace so the example stays isolated:

```bash
kubectl create namespace miel-demo
```

## Define the Deployment

A Deployment describes the desired state of our application. Kubernetes creates the Pods, replaces unhealthy instances, and coordinates rolling updates when the specification changes.

Create a file named `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: miel-web
  namespace: miel-demo
  labels:
    app: miel-web
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: miel-web
  template:
    metadata:
      labels:
        app: miel-web
    spec:
      containers:
        - name: web
          image: nginx:1.27-alpine
          ports:
            - name: http
              containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 200m
              memory: 128Mi
```

Several details here matter:

- The selector and Pod labels match, allowing the Deployment to manage the correct Pods.
- Three replicas give the application more than one running instance.
- The rolling update settings keep all existing replicas available while a replacement starts.
- The readiness probe prevents traffic from reaching a Pod before it is ready.
- Resource requests help the scheduler place Pods, while limits prevent one container from consuming unbounded resources.

## Expose the application

Pods are replaceable and their IP addresses can change. A Service gives the application a stable network identity and distributes traffic across ready Pods.

Create `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: miel-web
  namespace: miel-demo
spec:
  type: ClusterIP
  selector:
    app: miel-web
  ports:
    - name: http
      port: 80
      targetPort: http
```

`ClusterIP` exposes the service inside the cluster. In a real environment, an Ingress or Gateway would usually handle external traffic, TLS certificates, and routing.

## Apply and watch the rollout

Apply both manifests declaratively:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Do not stop at a successful `apply` message. Wait for Kubernetes to confirm that the rollout completed:

```bash
kubectl rollout status deployment/miel-web \
  --namespace miel-demo \
  --timeout 120s
```

Then inspect the resources:

```bash
kubectl get deployment,pods,service --namespace miel-demo
```

You should see three ready Pods, an available Deployment, and a Service with a cluster IP.

## Verify the application

For a local check, forward a port from your workstation to the Service:

```bash
kubectl port-forward service/miel-web 8080:80 \
  --namespace miel-demo
```

Open `http://localhost:8080` or test it from another terminal:

```bash
curl --fail http://localhost:8080
```

A successful NGINX response proves that the Service can route traffic to a ready Pod.

## Update and roll back safely

Change the container image through the Deployment rather than editing individual Pods:

```bash
kubectl set image deployment/miel-web \
  web=nginx:1.27.1-alpine \
  --namespace miel-demo

kubectl rollout status deployment/miel-web \
  --namespace miel-demo
```

If the new version fails its health checks or behaves incorrectly, inspect the history and roll back:

```bash
kubectl rollout history deployment/miel-web --namespace miel-demo
kubectl rollout undo deployment/miel-web --namespace miel-demo
```

## When the rollout fails

Start with the state Kubernetes already exposes:

```bash
kubectl get pods --namespace miel-demo
kubectl describe deployment miel-web --namespace miel-demo
kubectl describe pod <pod-name> --namespace miel-demo
kubectl logs <pod-name> --namespace miel-demo
kubectl get events --namespace miel-demo --sort-by=.lastTimestamp
```

Common causes include an image that cannot be pulled, an incorrect container port, failing probes, insufficient cluster capacity, and missing configuration or secrets. Read the Pod events before changing the manifest; they usually identify which layer is failing.

## Production considerations

This example establishes a sound workload baseline, but a production service usually needs more:

- Use immutable image digests instead of mutable tags.
- Add a PodDisruptionBudget and topology spread constraints for availability.
- Store application configuration in ConfigMaps and sensitive values in an appropriate secrets system.
- Define a HorizontalPodAutoscaler only after measuring realistic CPU, memory, or application metrics.
- Add an Ingress or Gateway with TLS and explicit network policies.
- Send metrics, logs, and traces to an observability platform.
- Validate manifests in CI and deploy them through GitOps or another audited delivery workflow.

The important habit is to treat the manifest as the beginning of the deployment—not proof that the application is working. Apply, observe, verify, and keep a tested recovery path.
