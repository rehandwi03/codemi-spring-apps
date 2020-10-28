# What you'll build

https://kubernetes.io[Kubernetes] is an open-source system for automating deployment, scaling, and management of containerized applications. It groups containers that make up an application into logical units for easy management and discovery. In this guide we will build and deploy a simple Spring boot application.

NOTE: There is also a https://spring.io/guides/gs/spring-boot-docker[Getting Started Guide] and a https://spring.io/guides/topicals/spring-boot-docker[Topical Guide] on Docker, which cover some of the background on building a container image.

== What you'll need
:java_version: 1.8
You will need a Linux or Linux-like command line. Command line examples in this guide work on Linux, a MacOS terminal with a shell, or https://docs.microsoft.com/en-us/windows/wsl[WSL] on Windows.

You will also need a Kubernetes cluster and the command line tool https://kubernetes.io/docs/tasks/tools/install-kubectl/[Kubectl]. You can create a cluster locally using https://github.com/kubernetes-sigs/kind[Kind] (on Docker) or https://github.com/kubernetes/minikube[Minikube]. Or you can use a cloud provider, such as https://console.cloud.google.com/kubernetes/[Google Cloud Platform], https://aws.amazon.com/eks/[Amazon Web Services] or https://azure.microsoft.com/en-gb/services/kubernetes-service/[Microsoft Azure]. Before proceeding further, verify you can run `kubectl` commands from the shell. E.g. (using `kind`):

```
$ kubectl cluster-info
Kubernetes master is running at https://127.0.0.1:46253
KubeDNS is running at https://127.0.0.1:46253/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

and

```
$ kubectl get all
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.43.0.1    <none>        443/TCP   7m13s
```

== Create a Spring Boot Application

The first thing we will do is create a Spring Boot application. If you have one you prefer to use already in github, you could clone it in the terminal (`git` and `java` are installed already). Or you can create an application from scratch using start.spring.io:

```
curl https://start.spring.io/starter.tgz -d dependencies=webflux,actuator | tar -xzvf -
```

You can then build the application:

```
./mvnw install
```

NOTE: It will take a couple of minutes the first time, but then once the dependencies are all cached it will be fast.

And you can see the result of the build. If the build was successful, you should see a JAR file, something like this:

```
ls -l target/*.jar
-rw-r--r-- 1 root root 19463334 Nov 15 11:54 target/demo-0.0.1-SNAPSHOT.jar
```

The JAR is executable:

```
$ java -jar target/*.jar
```

The app has some built in HTTP endpoints by virtue of the "actuator" dependency we added when we downloaded the project. So you will see something like this in the logs on startup:

```
...
2019-11-15 12:12:35.333  INFO 13912 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 2 endpoint(s) beneath base path '/actuator'
2019-11-15 12:12:36.448  INFO 13912 --- [           main] o.s.b.web.embedded.netty.NettyWebServer  : Netty started on port(s): 8080
...
```

So you can curl the endpoints in another terminal:

```
$ curl localhost:8080/actuator | jq .
{
  "_links": {
    "self": {
      "href": "http://localhost:8080/actuator",
      "templated": false
    },
    "health-path": {
      "href": "http://localhost:8080/actuator/health/{*path}",
      "templated": true
    },
    "health": {
      "href": "http://localhost:8080/actuator/health",
      "templated": false
    },
    "info": {
      "href": "http://localhost:8080/actuator/info",
      "templated": false
    }
  }
}
```

To complete this step, send Ctrl+C to kill the application.

== Containerize the Application

There are multiple options for containerizing a Spring Boot application. As long as you are already building a Spring Boot jar file, you only need to call the plugin directly. With https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/maven-plugin/html/#build-image[Maven]:

```
$ ./mvnw spring-boot:build-image
```

and with https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/gradle-plugin/reference/html/#build-image[Gradle]

```
$ ./gradlew bootBuildImage
```

You can run the container locally:

```
$ docker run -p 8080:8080 demo:0.0.1-SNAPSHOT
```

and check that it works in another terminal:

```
$ curl localhost:8080/actuator/health
```

Finish off by killing the container.

You won't be able to push the image unless you authenticate with Dockerhub (`docker login`), but there's an image there already that should work. If you were authenticated you could:

```
$ docker tag demo:0.0.1-SNAPSHOT springguides/demo
$ docker push springguides/demo
```

In real life the image needs to be pushed to Dockerhub (or some other accessible repository) because Kubernetes pulls the image from inside its Kubelets (nodes), which are not in general connected to the local docker daemon. For the purposes of this scenario you can omit the push and just use the image that is already there.

NOTE: Just for testing, there are workarounds that make `docker push` work with an insecure local registry, for instance, but that is out of scope for this guide.

== Deploy the Application to Kubernetes

You have a container that runs and exposes port 8080, so all you need to make Kubernetes run it is some YAML. To avoid having to look at or edit YAML, for now, you can ask `kubectl` to generate it for you. The only thing that might vary here is the `--image` name. If you deployed your container to your own repository, use its tag instead of this one:

```
$ kubectl create deployment demo --image=springguides/demo --dry-run -o=yaml > deployment.yaml
$ echo --- >> deployment.yaml
$ kubectl create service clusterip demo --tcp=8080:8080 --dry-run -o=yaml >> deployment.yaml
```

You can take the YAML generated above and edit it if you like, or you can just apply it:

```
$ kubectl apply -f deployment.yaml
deployment.apps/demo created
service/demo created
```

Check that the application is running:

```
$ kubectl get all
NAME                             READY     STATUS      RESTARTS   AGE
pod/demo-658b7f4997-qfw9l        1/1       Running     0          146m

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/kubernetes   ClusterIP   10.43.0.1       <none>        443/TCP    2d18h
service/demo         ClusterIP   10.43.138.213   <none>        8080/TCP   21h

NAME                   READY     UP-TO-DATE   AVAILABLE   AGE
deployment.apps/demo   1/1       1            1           21h

NAME                              DESIRED   CURRENT   READY     AGE
replicaset.apps/demo-658b7f4997   1         1         1         21h
d
```

TIP: Keep doing `kubectl get all` until the demo pod shows its status as "Running".

Now you need to be able to connect to the application, which you have exposed as a Service in Kubernetes. One way to do that, which works great at development time, is to create an SSH tunnel:

```
$ kubectl port-forward svc/demo 8080:8080
```

then you can verify that the app is running in another terminal:

```
$ curl localhost:8080/actuator/health
{"status":"UP"}
```
