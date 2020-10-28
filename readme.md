# What you'll need

You will need a Linux or Linux-like command line. Command line examples in this guide work on Linux, a MacOS terminal with a shell, or https://docs.microsoft.com/en-us/windows/wsl[WSL] on Windows.

You will also need a Docker Engine installed in your PC/Laptop, Kubernetes cluster and the command line tool https://kubernetes.io/docs/tasks/tools/install-kubectl/[Kubectl]. You can create a cluster locally using https://github.com/kubernetes-sigs/kind[Kind] (on Docker) or https://github.com/kubernetes/minikube[Minikube]. Or you can use a cloud provider, such as https://console.cloud.google.com/kubernetes/[Google Cloud Platform], https://aws.amazon.com/eks/[Amazon Web Services] or https://azure.microsoft.com/en-gb/services/kubernetes-service/[Microsoft Azure]. Before proceeding further, verify you can run `kubectl` commands from the shell. E.g. (using `kind`):

```
$ kubectl cluster-info
Kubernetes master is running at https://xx.xx.xx.xx
GLBCDefaultBackend is running at https://xx.xx.xx.xx/api/v1/namespaces/kube-system/services/default-http-backend:http/proxy
KubeDNS is running at https://xx.xx.xx.xx/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://xx.xx.xx.xx/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

and

```
$ kubectl get nodes
NAME                                       STATUS   ROLES    AGE     VERSION
gke-cluster-1-default-pool-f65e75f3-7f2h   Ready    <none>   4h17m   v1.16.13-gke.401
gke-cluster-1-default-pool-f65e75f3-kd41   Ready    <none>   4h17m   v1.16.13-gke.401
```

# Create a Docker images

Make sure you're in root project directory and run docker command:

```
$ docker build -t yourdockerimagename .
```

Example:

```
$ docker build -t 2017330017/codemi-spring-apps:v1 .
```

NOTE: Depending on your internet it will take a couple of minutes or more when create docker images, but then once the docker images are pulled it will be fast.

And you can see the result of the build. If the build was successful, you should see a docker images, something like this:

```
Successfully built 7c4f1d292c67
Successfully tagged 2017330017/codemi-spring-apps:v1
```

# Push a Docker image

After build the docker image, we'll push the image to the docker registry. Example, i'm using docker registry. If your docker registry is private, you must login with docker login command.

Docker login:

```
$ docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: yourusername
Password: yourpassword
```

Docker push image:

```
$ docker push yourimagename:tag
```

# Create Kubernetes Deployment with manifest file

In this step we'll create kubernetes deployment with manifest file. This is a manifest file including replicas, replicaset, rolling update, resources limits & requests and health check.

```
$ cat deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: codemi-spring-apps
  labels:
    role: app
spec:
  replicas: 2
  selector:
    matchLabels:
      role: app
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        role: app
    spec:
      containers:
      - image: 2017330017/codemi-spring-apps:v1
        name: spring-apps
        imagePullPolicy: Always
        resources:
          requests:
            cpu: "80m"
            memory: "100Mi"
          limits:
            cpu: "500m"
            memory: "650Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: http-port
          initialDelaySeconds: 90
          timeoutSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: http-port
          initialDelaySeconds: 60
          timeoutSeconds: 10
        ports:
        - containerPort: 8080
          name: http-port
      restartPolicy: Always
```

Create deployment with kubectl command:

```
$ kubectl apply -f deployment.yml
```

Check deployment and pods:

```
$ kubectl get deployment
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
codemi-spring-apps   2/2     2            2           4h30m
```

```
$ kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
codemi-spring-apps-86bfd48488-7zlbm   1/1     Running   0          9s
codemi-spring-apps-86bfd48488-h6hxc   1/1     Running   0          134m
```

# Create Kubernetes Services LoadBalancer with manifest file

In this step we'll create LoadBalancer service for codemi-spring-apps deployment. For binding service with deployment make sure the selector key:value is same as key:value in template.metadata.labels

```
$ cat service.yml
apiVersion: v1
kind: Service
metadata:
  name: codemi-spring-svc
spec:
  selector:
    role: app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
```

Create kubernetes service with command:

```
$ kubectl apply -f service.yml
```

Check kubernetes service with command:

```
$ kubectl get services
NAME                TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)        AGE
codemi-spring-svc   LoadBalancer   10.8.0.25    xx.xxx.xxx.xxx   80:31091/TCP   3h15m
kubernetes          ClusterIP      10.8.0.1     <none>           443/TCP        5h5m
```

Check service with curl:

```
$ curl http://xx.xxx.xxx.xxx
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Web Services</title>
  </head>
  <body>
    <h1>Codemi Sample Apps</h1>
  </body>
</html>
```

# Create Horizontal Pod Autoscaler

In this step we'll create horizontal pod autoscaler for automaticly scaling pods when traffic requests to pods is high or low. For binding hpa with deployment make sure in scaleTargetRef.name is same as the deployment name.

```
$ cat hpa.yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: codemi-spring-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: codemi-spring-apps
  minReplicas: 1
  maxReplicas: 2
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 85
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 600Mi
```

Create hpa with command:

```
$ kubectl apply -f hpa.yaml
```

Check the hpa:

```
$ kubectl get hpa
NAME                REFERENCE                       TARGETS                   MINPODS   MAXPODS   REPLICAS   AGE
codemi-spring-hpa   Deployment/codemi-spring-apps   111771648/600Mi, 2%/85%   1         2         1          165m
```

# Check all resources in default namespace in kubernetes

```
$ kubectl get all
NAME                                      READY   STATUS    RESTARTS   AGE
pod/codemi-spring-apps-86bfd48488-h6hxc   1/1     Running   0          167m

NAME                        TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)        AGE
service/codemi-spring-svc   LoadBalancer   10.8.0.25    xx.xxx.xxx.xxx   80:31091/TCP   3h33m
service/kubernetes          ClusterIP      10.8.0.1     <none>           443/TCP        5h23m

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/codemi-spring-apps   1/1     1            1           5h5m

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/codemi-spring-apps-6b78f9f858   0         0         0       171m
replicaset.apps/codemi-spring-apps-789fc7f98d   0         0         0       5h5m
replicaset.apps/codemi-spring-apps-84f558b6f9   0         0         0       3h33m
replicaset.apps/codemi-spring-apps-86bfd48488   1         1         1       167m

NAME                                                    REFERENCE                       TARGETS                   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/codemi-spring-hpa   Deployment/codemi-spring-apps   111771648/600Mi, 2%/85%   1         2         1          167m
```
