+++
title = 'Déployer une application sur Kubernetes'
description = 'Un guide pratique pour livrer une application web conteneurisée avec un Deployment et un Service, puis vérifier le déploiement en toute sécurité.'
summary = 'Construisez un déploiement Kubernetes modeste mais pensé pour la production : déclarez la charge de travail, exposez-la avec un Service, suivez le déploiement et sachez où chercher en cas d’échec.'
date = '2026-08-02T09:00:00+01:00'
draft = false
authors = ['Mohamed Lembarki', 'Ihsane Elkhaldi']
topics = ['Kubernetes', 'Conteneurs', 'DevOps']
+++

Les déploiements Kubernetes deviennent beaucoup plus simples à comprendre lorsque l’on sépare le processus en trois questions : **qu’est-ce qui doit s’exécuter, comment Kubernetes doit-il le maintenir en bonne santé, et comment le trafic doit-il l’atteindre ?** Dans ce guide, nous allons déployer une petite application web NGINX avec un `Deployment` et un `Service`.

La même structure s’applique à la plupart des applications sans état. Remplacez l’image, les ports, les vérifications de santé et les valeurs de ressources par ceux dont votre service a besoin.

## Avant de commencer {#before-you-begin}

Vous avez besoin d’un cluster Kubernetes fonctionnel et de `kubectl` configuré pour communiquer avec lui. Un cluster local comme kind ou minikube suffit pour cet exemple.

Vérifiez que la connexion fonctionne :

```bash
kubectl cluster-info
kubectl get nodes
```

Créez un espace de noms dédié afin de garder l’exemple isolé :

```bash
kubectl create namespace miel-demo
```

## Définir le Deployment {#define-the-deployment}

Un Deployment décrit l’état souhaité de notre application. Kubernetes crée les Pods, remplace les instances défaillantes et coordonne les mises à jour progressives lorsque la spécification change.

Créez un fichier nommé `deployment.yaml` :

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

Plusieurs détails comptent ici :

- Le sélecteur et les labels des Pods correspondent, ce qui permet au Deployment de gérer les bons Pods.
- Trois réplicas donnent à l’application plus d’une instance en fonctionnement.
- Les réglages de mise à jour progressive gardent les réplicas existants disponibles pendant qu’un remplacement démarre.
- La sonde de disponibilité empêche le trafic d’atteindre un Pod avant qu’il soit prêt.
- Les demandes de ressources aident le planificateur à placer les Pods, tandis que les limites empêchent un conteneur de consommer des ressources sans borne.

## Exposer l’application {#expose-the-application}

Les Pods sont remplaçables et leurs adresses IP peuvent changer. Un Service donne à l’application une identité réseau stable et répartit le trafic entre les Pods prêts.

Créez `service.yaml` :

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

`ClusterIP` expose le service à l’intérieur du cluster. Dans un environnement réel, un Ingress ou une Gateway gère généralement le trafic externe, les certificats TLS et le routage.

## Appliquer et suivre le déploiement {#apply-and-watch-the-rollout}

Appliquez les deux manifestes de manière déclarative :

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Ne vous arrêtez pas à un message `apply` réussi. Attendez que Kubernetes confirme que le déploiement est terminé :

```bash
kubectl rollout status deployment/miel-web \
  --namespace miel-demo \
  --timeout 120s
```

Inspectez ensuite les ressources :

```bash
kubectl get deployment,pods,service --namespace miel-demo
```

Vous devriez voir trois Pods prêts, un Deployment disponible et un Service avec une adresse IP de cluster.

## Vérifier l’application

Pour une vérification locale, redirigez un port de votre poste vers le Service :

```bash
kubectl port-forward service/miel-web 8080:80 \
  --namespace miel-demo
```

Ouvrez `http://localhost:8080` ou testez depuis un autre terminal :

```bash
curl --fail http://localhost:8080
```

Une réponse NGINX réussie prouve que le Service peut acheminer le trafic vers un Pod prêt.

## Mettre à jour et revenir en arrière en sécurité

Modifiez l’image du conteneur via le Deployment plutôt qu’en éditant des Pods individuellement :

```bash
kubectl set image deployment/miel-web \
  web=nginx:1.27.1-alpine \
  --namespace miel-demo

kubectl rollout status deployment/miel-web \
  --namespace miel-demo
```

Si la nouvelle version échoue à ses vérifications de santé ou se comporte mal, inspectez l’historique et revenez en arrière :

```bash
kubectl rollout history deployment/miel-web --namespace miel-demo
kubectl rollout undo deployment/miel-web --namespace miel-demo
```

## En cas d’échec du déploiement {#when-the-rollout-fails}

Commencez par l’état que Kubernetes expose déjà :

```bash
kubectl get pods --namespace miel-demo
kubectl describe deployment miel-web --namespace miel-demo
kubectl describe pod <pod-name> --namespace miel-demo
kubectl logs <pod-name> --namespace miel-demo
kubectl get events --namespace miel-demo --sort-by=.lastTimestamp
```

Les causes fréquentes comprennent une image impossible à récupérer, un mauvais port de conteneur, des sondes défaillantes, une capacité de cluster insuffisante et des configurations ou secrets manquants. Lisez les événements des Pods avant de modifier le manifeste : ils indiquent généralement quelle couche échoue.

## Considérations pour la production {#production-considerations}

Cet exemple établit une bonne base pour la charge de travail, mais un service de production a généralement besoin de davantage :

- Utilisez des digests d’images immuables plutôt que des tags modifiables.
- Ajoutez un PodDisruptionBudget et des contraintes de répartition topologique pour la disponibilité.
- Stockez la configuration applicative dans des ConfigMaps et les valeurs sensibles dans un système de secrets approprié.
- Définissez un HorizontalPodAutoscaler uniquement après avoir mesuré des métriques CPU, mémoire ou applicatives réalistes.
- Ajoutez un Ingress ou une Gateway avec TLS et des politiques réseau explicites.
- Envoyez métriques, journaux et traces vers une plateforme d’observabilité.
- Validez les manifestes dans la CI et déployez-les avec GitOps ou un autre mécanisme de livraison audité.

L’habitude importante est de considérer le manifeste comme le début du déploiement, et non comme la preuve que l’application fonctionne. Appliquez, observez, vérifiez et conservez un chemin de reprise testé.
