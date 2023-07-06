
## - Ingress

- Download the lesson folder from github.

The directory structure is as follows:

```text
ingress-yaml-files
├── ingress-service.yaml
├── php-apache
│   └── php-apache.yaml
└── to-do
    ├── db-deployment.yaml
    ├── db-pvc.yaml
    ├── db-service.yaml
    ├── web-deployment.yaml
    └── web-service.yaml
```

- Alternatively you can clone some part of your repository as show below:

```shell
sudo yum install git -y
mkdir repo && cd repo
git init
git remote add origin <origin-url>
git config core.sparseCheckout true
echo "subdirectory/under/repo/" >> .git/info/sparse-checkout  # do not put the repository folder name in the beginning
git pull origin <branch-name>
```

### Steps of execution:

1. We will deploy the `to-do` app first and look at some key points.
2. And then deploy the `php-apache` app and highlights some important points.
3. We will introduce the `ingress-service` and talk about it.

Let's check the state of the cluster and see that everything works fine.

```bash
kubectl cluster-info
kubectl get node
```

- Go to the `volume-and-ingress-yaml-files/to-do` directory and look at the contents.

Let's check the MongoDB `service`.

```bash
cat db-service.yaml
```
- You will see an output like this

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-service
  labels:
    name: mongo
    app: todoapp
spec:
  selector:
    name: mongo
  type: ClusterIP
  ports:
    - name: db
      port: 27017
      targetPort: 27017
```

Note that a database has no direct exposure the outside world, so it's type is `ClusterIP`.

Now check the content of the front-end web application `service`.

```bash
cat web-service.yaml
```
- You will see an output like this

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  labels:
    name: web
    app: todoapp
spec:
  selector:
    name: web 
  type: LoadBalancer
  ports:
   - name: http
     port: 3000
     targetPort: 3000
     protocol: TCP
```
What should be the type of the service? ClusterIP, NodePort or LoadBalancer?

Check the web application `Deployment` file.
```bash
cat web-deployment.yaml
```
- You will see an output like this

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      name: web
  template:
    metadata:
      labels:
        name: web
        app: todoapp
    spec:
      containers: 
        - image: clarusways/todo
          imagePullPolicy: Always
          name: myweb
          ports: 
            - containerPort: 3000
          env:
            - name: "DBHOST"
              value: "db-service:27017"
          resources:
            limits:
              cpu: 100m
            requests:
              cpu: 80m
```

Let's deploy the to-do application.

```bash
cd ..
kubectl apply -f to-do
deployment.apps/db-deployment created
persistentvolumeclaim/database-persistent-volume-claim created
service/db-service created
deployment.apps/web-deployment created
service/web-service created
```
Note that we can use `directory` with `kubectl apply -f` command.

- Check the pods.
```bash
kubectl get pods
```
- You will see an output like this

```text
NAME                              READY   STATUS    RESTARTS   AGE
db-deployment-8597967796-q7x5s    1/1     Running   0          4m30s
web-deployment-658cc55dc8-2h2zc   1/1     Running   2          4m30s
```

- Check the services.
```bash
kubectl get svc
```
- You will see an output like this

```text
NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)          AGE
db-service           ClusterIP      10.100.199.214   <none>                                                                    27017/TCP        22m
kubernetes           ClusterIP      10.100.0.1       <none>                                                                    443/TCP          120m
web-service          LoadBalancer   10.100.59.43     a2a513b28b46b4a20848f8303294e90f-1926642410.us-east-2.elb.amazonaws.com   3000:31860/TCP   22m
```
Note the `PORT(S)` difference between `db-service` and `web-service`. Why?

- We can visit a2a513b28b46b4a20848f8303294e90f-1926642410.us-east-2.elb.amazonaws.com:3000 and access the application.

or

```bash
curl a2a513b28b46b4a20848f8303294e90f-1926642410.us-east-2.elb.amazonaws.com:3000 
OK!
```
We see the home page. You can add to-do's.

- Now deploy the second application

```bash
cd php-apache/
cat php-apache.yaml
```
- You will see an output like this

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 100m
          requests:
            cpu: 80m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache-service
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache 
  type: LoadBalancer	
```

Note how the `Deployment` and `Service` `yaml` files are merged in one file. 

- Deploy this `php-apache` file.

```bash
kubectl apply -f php-apache.yaml 
```

- Get the pods.

```bash
kubectl get po
```
- You will see an output like this
```text
NAME                              READY   STATUS    RESTARTS   AGE
db-deployment-8597967796-q7x5s    1/1     Running   0          17m
php-apache-7869bd4fb-xsvnh        1/1     Running   0          24s
web-deployment-658cc55dc8-2h2zc   1/1     Running   2          17m
```

- Get the services.

```bash
kubectl get svc
```
- You will see an output like this

```text
NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)          AGE
db-service           ClusterIP      10.100.199.214   <none>                                                                    27017/TCP        22m
kubernetes           ClusterIP      10.100.0.1       <none>                                                                    443/TCP          120m
php-apache-service   LoadBalancer   10.100.191.10    ac4c071f935d64c3cb535e87e50c8216-186981612.us-east-2.elb.amazonaws.com    80:31850/TCP     59m
web-service          LoadBalancer   10.100.59.43     a2a513b28b46b4a20848f8303294e90f-1926642410.us-east-2.elb.amazonaws.com   3000:31860/TCP   22m
```

Let's check what web app presents us.

- On opening browser (ac4c071f935d64c3cb535e87e50c8216-186981612.us-east-2.elb.amazonaws.com ) we see

```text
OK!
```

Alternatively, you can use;
```bash
curl ac4c071f935d64c3cb535e87e50c8216-186981612.us-east-2.elb.amazonaws.com 
OK!
```

## Ingress

Briefly explain ingress and ingress controller. For additional information a few portal can be visited like;

- https://kubernetes.io/docs/concepts/services-networking/ingress/
  
- https://banzaicloud.com/blog/k8s-ingress/
  
- Open the offical [ingress-nginx]( https://kubernetes.github.io/ingress-nginx/deploy/ ) explain the `ingress-controller` installation steps for different architecture.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.3.0/deploy/static/provider/cloud/deploy.yaml
```

- Now, check the contents of the `ingress-service`.

```bash
 cat ingress-service.yaml
```
- You will see an output like this

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-service
  annotations:
    kubernetes.io/ingress.class: 'nginx'
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port: 
                  number: 3000
          - path: /load
            pathType: Prefix
            backend:
              service:
                name: php-apache-service
                port: 
                  number: 80
```

- Explain the rules part.

```bash
kubectl apply -f ingress-service.yaml
```
- You will see an output like this

```bash
kubectl get ingress
NAME              HOSTS   ADDRESS                                                                            PORTS   AGE
ingress-service   *       a26be57ce12e64883a5ad050025f2c5b-94ab4c4b033cf5fa.elb.eu-central-1.amazonaws.com   80      2m8s
```

On browser, type this  ( a26be57ce12e64883a5ad050025f2c5b-94ab4c4b033cf5fa.elb.eu-central-1.amazonaws.com ), you must see the to-do app web page. If you type `a26be57ce12e64883a5ad050025f2c5b-94ab4c4b033cf5fa.elb.eu-central-1.amazonaws.com/load`, then the apache-php page, "OK!". Notice that we don't use the exposed ports at the services.

- Delete the cluster

```bash
eksctl get cluster --region us-east-1
```
- You will see an output like this

```text
NAME            REGION
cw-cluster      us-east-1
```
```bash
eksctl delete cluster cw-cluster --region us-east-1
```

- Do no forget to delete related ebs volumes.
