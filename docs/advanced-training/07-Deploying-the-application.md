# 06 - Deploying the application

Now that we have our Kubernetes cluster ready, let's make use of it and
deploy our application. The resources needed for the deployment are
already prepared. Let's start by taking a look at a few basic Kubernetes
objects:

- **_Pod_**: this is the most basic object in Kubernetes. A pod represents
  processes running on the cluster. It encapsulates one or more
  containers, contains storage and has a unique network IP. A specific
  Pod always runs on one specific node of the cluster. Pods can be
  created and managed by Controllers. One of these Controllers is a
  Deployment.
- **_Deployment_**: you can describe the desired state of Pods in a
  Deployment object and the Deployment controller will take all
  necessary steps to reach that state. A Deployment can be used to
  describe which containers you want to run inside your Pod and how
  many copies of your Pod you want to run in parallel.
- **_Service_**: a service defines a logical set of Pods and a policy by how
  to access them. You can for example specify on which ports you want
  your Pods to be available or that you want to use a LoadBalancer to
  access your Pods.

We already put all our objects that we need for our application into a file called `/devops-training-application/deploy/application.yaml`, so you don't need to do that.
But to get a better understanding of the individual objects, we are going to look at it piece by piece:

**_devops-training-application/deploy/application.yaml_**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: voting
spec:
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: <your_ECR_repositoryUri>
          resources:
            limits:
              memory: '128Mi'
              cpu: '100m'
          ports:
            - containerPort: 8080
          env:
            - name: REDIS_SERVER
              value: 'redis.voting.svc.cluster.local'
            - name: REDIS_PORT
              value: '6379'
            - name: VOTE_VALUE_1
              value: 'CATS'
            - name: VOTE_VALUE_2
              value: 'DOGS'
```

- in line 7, we specify that we want a Deployment
- the metadata block in lines 8 - 10 describes the name of the
  Deployment and the namespace it lives in
- the spec block in lines 11 and following specifies the container
  that we want to run inside this Pod - this is the frontend
  application that we dockerized and pushed to our ECR repository. You
  need to point to the ECR repository that you created earlier here.
  You can get the `repositoryUri` from the output of:

  ```bash
  aws ecr describe-repositories --region eu-central-1
  ```

- in line 28, we specify the port that our application is available
  at. We are going to make that port available using our Kubernetes
  Service, which looks like this:

**_devops-training-application/deploy/application.yaml_**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: voting
spec:
  selector:
    app: frontend
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
```

- We specify a Service with a selector. That way, any traffic coming
  in to that service will be routed to one of the Pods with the label
  `app: frontend`.
- The service will be exposed as type LoadBalancer (line 50), which will automatically create an AWS LoadBalancer for it

Let's take a look at the Redis deployment:

**_devops-training-application/deploy/application.yaml_**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: voting
spec:
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis
          resources:
            limits:
              memory: '128Mi'
              cpu: '500m'
          ports:
            - containerPort: 6379
          command:
            - redis-server
            - '/redis-master/redis.conf'
          volumeMounts:
            - mountPath: /redis-master
              name: config
      volumes:
        - name: config
          configMap:
            name: redis-config
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: voting
spec:
  selector:
    app: redis
  ports:
    - port: 6379
```

- We create a Deployment in the same namespace as the frontend
  application
- The interesting part here is the image specification in line 83 - it
  just says `redis`. Remember that we specified our own Docker
  registry and image name for the frontend above? If you don't specify
  a Docker registry, Kubernetes will look for the image on the default
  registry - which is Dockerhub. So we are simply using a default
  image for Redis.
- We also mount a volume into our container which contains a file called `redis.conf`. This is a Redis-specific configuration file, where we limit the maximum memory that Redis is allowed to use.
- Our Service does not expose our Redis with a LoadBalancer, since we don't
  want it to be open to the world. Only our frontend container needs
  to be able to talk to it.

## Deploying the application

Deploying the application is really simple. We specified all resources
in a declarative way. Now we are going to hand this file over to
Kubernetes and let it do the necessary work to create all objects for
us:

```bash
cd ~/devops-training/devops-training-application/deploy
kubectl apply -f application.yaml
```

The output shows us which objects were created:

```bash
namespace/voting created
deployment.apps/frontend created
service/frontend created
deployment.apps/redis created
service/redis created
```

## Accessing the application

You can check the status of our created Pods to see if everything is
ready:

```bash
kubectl get pods -n voting
```

Output:

```bash
NAME                        READY   STATUS    RESTARTS   AGE
frontend-5fc4d86d7f-ckmjj   1/1     Running   0          17h
redis-86b78c7b7b-spzz7      1/1     Running   0          17h
```

When the status shows `Running`, we can use our application. Remember that we exposed the Node.js application with a LoadBalancer? Now we need to get the DNS name of that LoadBalancer. Run the following command to get it:

```bash
kubectl get services -n voting
```

The output shows the DNS name of the LoadBalancer in the column `EXTERNAL-IP`:

```bash
NAME       TYPE           CLUSTER-IP       EXTERNAL-IP                                                                  PORT(S)        AGE
frontend   LoadBalancer   172.20.251.160   ab7e5cdf1a2ec11e98b7b0217677ec25-1471509914.eu-central-1.elb.amazonaws.com   80:31530/TCP   108m
redis      ClusterIP      172.20.44.54     <none>                                                                       6379/TCP       108m
```

Now you can access your application from your browser by opening the URL of the LoadBalancer.
