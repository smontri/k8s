# Lab 2: Deploy PHP Guestbook application with Redis

## Start up the Redis master 

The guestbook application uses Redis to store its data. It writes its data to a Redis master instance and reads data from multiple Redis slave instances.

### Creating the Redis Master Deployment

The manifest file, included below, specifies a Deployment controller that runs a single replica Redis master Pod.

```yaml
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: redis-master
  labels:
    app: redis
spec:
  selector:
    matchLabels:
      app: redis
      role: master
      tier: backend
  replicas: 1
  template:
    metadata:
      labels:
        app: redis
        role: master
        tier: backend
    spec:
      containers:
      - name: master
        image: k8s.gcr.io/redis:e2e  # or just image: redis
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 6379
```

**Apply the Redis Master Deployment from the `redis-master-deployment.yaml` file:**

```shell
kubectl apply -f https://k8s.io/examples/application/guestbook/redis-master-deployment.yaml -n <your-namespace>
```
	
**Query the list of Pods to verify that the Redis Master Pod is running:**

```shell
kubectl get pods -n <your-namespace>
```
![Redis Master Deploy](images/lab2_deploy_redis_master.png)

### Creating the Redis Master Service

The guestbook applications needs to communicate to the Redis master to write its data. You need to apply a Service to proxy the traffic to the Redis master Pod. A Service defines a policy to access the Pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-master
  labels:
    app: redis
    role: master
    tier: backend
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
    role: master
    tier: backend
```

**Apply the Redis Master Service from the `redis-master-service.yaml` file:**

```shell
kubectl apply -f https://k8s.io/examples/application/guestbook/redis-master-service.yaml -n <your-namespace>
```
![Redis Master Service](images/lab2_service_redis_master.png)

You can check the logs of the Redis Master pod with the command:

```shell
kubectl logs -f <POD-NAME>
```
![Redis Master logs](images/lab2_redis_master_logs.png)

## Start up the Redis Slaves

### Creating the Redis Slave Deployment

Deployments scale based off of the configurations set in the manifest file. In this case, the Deployment object specifies two replicas.

If there are not any replicas running, this Deployment would start the two replicas on your container cluster. Conversely, if there are more than two replicas are running, it would scale down until two replicas are running.

```yaml
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: redis-slave
  labels:
    app: redis
spec:
  selector:
    matchLabels:
      app: redis
      role: slave
      tier: backend
  replicas: 2
  template:
    metadata:
      labels:
        app: redis
        role: slave
        tier: backend
    spec:
      containers:
      - name: slave
        image: gcr.io/google_samples/gb-redisslave:v3
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # Using `GET_HOSTS_FROM=dns` requires your cluster to
          # provide a dns service. As of Kubernetes 1.3, DNS is a built-in
          # service launched automatically. However, if the cluster you are using
          # does not have a built-in DNS service, you can instead
          # access an environment variable to find the master
          # service's host. To do so, comment out the 'value: dns' line above, and
          # uncomment the line below:
          # value: env
        ports:
        - containerPort: 6379
```

**Apply the Redis Slave Deployment from the `redis-slave-deployment.yaml` file:**

```shell
kubectl apply -f https://k8s.io/examples/application/guestbook/redis-slave-deployment.yaml -n <your-namespace>
```
![Redis Slave Pods](images/lab2_redis_slave_deploy.png)

### Creating the Redis Slave Service

The guestbook application needs to communicate to Redis slaves to read data. To make the Redis slaves discoverable, you need to set up a Service. A Service provides transparent load balancing to a set of Pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-slave
  labels:
    app: redis
    role: slave
    tier: backend
spec:
  ports:
  - port: 6379
  selector:
    app: redis
    role: slave
    tier: backend
```

**Apply the Redis Slave Sercice from the `redis-slave-service.yaml` file:**

```shell
kubectl apply -f https://k8s.io/examples/application/guestbook/redis-slave-service.yaml -n <your-namespace>
```
![Redis Slave Service](images/lab2_service_redis_slave.png)

## Set Up and Expose the Guestbook Frontend

The guestbook application has a web frontend serving the HTTP requests written in PHP. It is configured to connect to the `redis-master` Service for write requests and the `redis-slave` service for Read requests.

### Creating the Guestbook Frontend Deployment

```yaml
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: frontend
  labels:
    app: guestbook
spec:
  selector:
    matchLabels:
      app: guestbook
      tier: frontend
  replicas: 3
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google-samples/gb-frontend:v4
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # Using `GET_HOSTS_FROM=dns` requires your cluster to
          # provide a dns service. As of Kubernetes 1.3, DNS is a built-in
          # service launched automatically. However, if the cluster you are using
          # does not have a built-in DNS service, you can instead
          # access an environment variable to find the master
          # service's host. To do so, comment out the 'value: dns' line above, and
          # uncomment the line below:
          # value: env
        ports:
        - containerPort: 80
```

**Apply the Frontend Deployment from the `frontend-deployment.yaml` file:**

```shell
kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-deployment.yaml -n <your-namespace>
```

And query list of Pods to verify that the three frontend replicas are running

```shell
kubectl get pods -l app=guestbook -l tier=frontend -n <your-namespace>
```

![Frontend Deploy](images/lab2_frontend_deploy.png)

### Creating the Frontend Service

The `redis-slave` and `redis-master` Services you applied are only accessible within the container cluster because the default type for a Service is `ClusterIP`. ClusterIP provides a single IP address for the set of Pods the Service is pointing to. This IP address is accessible only within the cluster.

If you want guests to be able to access your guestbook, you must configure the frontend Service to be externally visible, so a client can request the Service from outside the container cluster. We will expose the Frontend Service through `NodePort`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # comment or delete the following line if you want to use a LoadBalancer
  type: NodePort 
  # if your cluster supports it, uncomment the following to automatically create
  # an external load-balanced IP for the frontend service.
  # type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: guestbook
    tier: frontend
```

**Apply the Frontend Service from the `frontend-service.yaml` file:**

```shell
kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-service.yaml -n <your-namespace>
```
![Frontend Service](images/lab2_service_frontend.png)

### Accessing the application Guestbook

Exposing an application through NodePort, enables the access to the application from any of the nodes of the cluster by using the port defined in the service.

![Accessing application](images/app_master.png)

THIS ENDS LAB #2!