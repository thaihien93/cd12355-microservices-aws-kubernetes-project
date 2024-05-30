# Coworking Space Service
The Coworking Space Service is a set of APIs that enables users to request one-time tokens and administrators to authorize access to a coworking space. This service follows a microservice pattern and the APIs are split into distinct services that can be deployed and managed independently of one another.

## Getting Started

### 1. Create an EKS Cluster
#### - Ensure the AWS CLI is configured correctly.
```bash
$ aws sts get-caller-identity
```

#### - Create an EKS Cluster
```bash
$ eksctl create cluster --name hient4-cluster --region us-east-1 --nodegroup-name hient4-nodes --node-type m6g.large --nodes 1 --nodes-min 1 --nodes-max 2
```

#### - Update the Kubeconfig
```bash
$ aws eks --region us-east-1 update-kubeconfig --name hient4-cluster
$ kubectl config current-context
```

#### - Alternatively, view the entire Kubeconfig file
```bash
$ kubectl config view
```

---

### 2. Configure a Database
#### - Create PersistentVolumeClaim
Create a file pvc.yaml on your local machine, with the following content.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

#### - Create PersistentVolume
Create a file pv.yaml on your local machine, with the following content.
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-manual-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: gp2
  hostPath:
    path: "/mnt/data"
```

#### - Create Postgres Deployment
Create a file postgresql-deployment.yaml on your local machine, with the following content.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
spec:
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
      - name: postgresql
        image: postgres:latest
        env:
        - name: POSTGRES_DB
          value: mydatabase
        - name: POSTGRES_USER
          value: myuser
        - name: POSTGRES_PASSWORD
          value: mypassword
        ports:
        - containerPort: 5432
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: postgresql-storage
      volumes:
      - name: postgresql-storage
        persistentVolumeClaim:
          claimName: postgresql-pvc
```

#### - Run Apply Commands
```bash
$ cd /deployment
$ kubectl apply -f configmap.yaml

$ cd /database
$ kubectl apply -f pvc.yaml
$ kubectl apply -f pv.yaml
$ kubectl apply -f postgresql-deployment.yaml
```

#### - Test Database Connection
- View the pods
```bash
$ kubectl get pods
```

- Run the following command to open bash into the pod.
```bash
$ kubectl exec -it <POD-POSGRESQL-NAME> -- bash
$ psql -U myuser -d mydatabase
```

- Connecting via Port Forwarding
Create a YAML file, postgresql-service.yaml, in your current working directory.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgresql-service
spec:
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: postgresql
```
Then apply this file:
```bash
$ kubectl apply -f postgresql-service.yaml
```

- When you have exited the pod and arrived back to your local environment, run the following (trivia: svc and service are interchangeable):
```bash
# List the services
$ kubectl get svc

# Set up port-forwarding to `postgresql-service`
$ kubectl port-forward service/postgresql-service 5433:5432 &
```

#### - Run Seed Files & Checking the tables
- Run the above command once for each SQL file in the db directory. Alternatively, you may keep the commands in an initialization bash script.
```bash
$ cd db/
$ PGPASSWORD="$DB_PASSWORD" psql --host 127.0.0.1 -U myuser -d mydatabase -p 5433 < <FILE_NAME.sql>
```

- Run the following command to open up the psql terminal to ensure they are not empty.
```bash
$ PGPASSWORD="$DB_PASSWORD" psql --host 127.0.0.1 -U myuser -d mydatabase -p 5433

select *from users;
select *from tokens;
```

---

### 3. Deploy the Analytics Application
#### - Dockerize the Application
- You just need to write the Dockerfile for the coworking analytics application. For my project, I stored it inside the `analytics/` directory.
```yaml
#!/bin/bash

FROM python:3.11-slim

WORKDIR /usr/src/app

COPY . .

RUN apt update -y && \
    pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5153

CMD [ "python", "./app.py" ]
```

#### - Set up Continuous Integration (CI) with CodeBuild
- First, create an Amazon ECR repository on your AWS console.
- Then, create an Amazon CodeBuild project that is connected to your project's GitHub repository.
- Once they are done, create a buildspec.yaml file that will be triggered whenever the project repository is updated
`The relevant course content for this task is available in the "Microservices for DevOps on AWS" lesson.`

#### - Deploy the Application
Run the commands
```bash
$ cd deployment/
$ kubectl apply -f coworking.yaml
```

Verify the Deployment
```bash
$ kubectl get pods
```
It's done when you see your `coworking` pod have status is `Running` and Ready is `1/1`

Get Endpoint after deploy
```bash
$ kubectl get svc
```
After that, copy EXTERNAL-IP of NAME `coworking` and paste it to browser like this: `http://a8b2500cf1db0430bab3be86103f43a0-1715846082.us-east-1.elb.amazonaws.com:5153/health_check`. When you see response is `ok`, you dit it!

---

## Review the Project Rubric
Review again the Project Rubric to be sure you're not miss anything.