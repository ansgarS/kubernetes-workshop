# kubernetes-workshop

An introduction into Kubernetes for Beginners - slides can be found [here](./Ansgar%20Sachs%20-%20Kubernetes%20Workshop.pdf).

![K8S Workshop Intro Slide](./k8s_intro_slide.png)

## 1 Branches

The repository splits up into different branches depending on your current progress.

| Task | Branch        | Description                                           |
| ---- | ------------- | ----------------------------------------------------- |
| 1    | master        | Initial skeleton project                              |
| 1    | task1/final   | Implements a monolith to serve all requested routes   |
| 2    | task2/starter | Initial split up into services                        |
| 2    | task2/final   | Implements two services to serve all requested routes |
| 3    | task3/starter | Initial kubernetes files                              |
| 3    | task3/final   | Deploys all services to a k8s cluster                 |

## 2 Tasks

### 2.1 Create a monolith

Build a monolithic application that serves the following aspects:

1) Create new patients
2) Create new prescriptions
3) List all patient ids
4) Calculate the costs of prescriptions for a given patient

You can use the integration tests for validating your artifacts.
In order to continue with the next task more easily, you should 
create different entities for your read and write model. Please
keep in mind that the entity IDs are generated on the server and 
are returned wrapped in an object - not only a simple string.

The task doesn't require any database access - instead you should
persist everything in an in-memory store (hint: singleton).

The directory `patient-monolith` is the only place you need to touch 
for this task.

The following routes have to be served:

```
POST /patients {
  "firstName": "...",
  "lastName": "...",
  "age": 123
}
Returns 201 Created { "id":  "..." }

POST /prescriptions {
  "patientId": "...",
  "name": "...",
  "dose": "...",
  "price": 12.12
}
Returns 201 Created { "id": "..." }


GET /patients/<id>/costs

Returns 200 OK { "patientId": "...", "costs": 45.27 }
Returns 404 Not found if patient does not exist

GET /patients 

Returns 200 OK {
  "patientIds": ["id1", "id2"]
}
```

Note: Do not remove the dependency `spring-boot-starter-actuator`. 
It is used for health checks during integration tests.

Hint: How to create a rest controller

```java
@RestController
@RequestMapping(value = "/myuri")
public class SomeController {
    // ....

    @PostMapping(produces = {"application/json"})
    public ResponseEntity<EntityId> createSth(@RequestBody MyEntity entity) {
        try {
            // ...

            return new ResponseEntity<>(entityId, HttpStatus.CREATED);
        } catch (SomeException e) {
            return new ResponseEntity<>(HttpStatus.UNPROCESSABLE_ENTITY);
        }
    }
}
```

### 2.2 Split into services and containerize them

Split the write and the read concern of your monolithic application into two services and 
create docker containers that are composed inside docker-compose.

Before your start, merge the branch `task2/starter` into your current branch. 
As a result, you should see the following additional artifacts:
1) insurance-service: Service that lists patientIds and calculates prescription costs
2) patient-service: Service that creates patients and prescriptions
3) docker-compose.yml: Composes a postgres database and both services 

This time you have to take care of the business logic in both services - the patient-monolith 
is not relevant anymore. In addition, you have to think about Dockerfiles that serve your spring
application - don't make it a rocket science and use the internet for inspiration.
However, as a first step, you should create tables for your views and 
additional materialized view which will fake eventual consistency.
For this purpose have a look into [init.sql](./patient-db/init.sql)
and add the following:

1) Patient Table
2) Prescription Table
3) Dispenses Materialized View
4) PatientIds Materialized View

Hint: You need to add triggers which update the materialized view on
table updates. Materialized views are used for this task to separate 
the READ database from the WRITE database. Such a view can be 
implemented as follows (or read the documentation [here](https://www.postgresql.org/docs/current/rules-materializedviews.html)):

```sql
--  patient-db/init.sql
CREATE TABLE test_entity (
      id          varchar(255) PRIMARY KEY,
      first_name  varchar(255) NOT NULL
);

CREATE MATERIALIZED VIEW IF NOT EXISTS test_ids AS
-- Here you need to aggregate with a smart select query.
-- You can use aggregation functions like sum() and 
-- GROUP BY another table column
SELECT
    id
FROM test_entity;

CREATE UNIQUE INDEX IF NOT EXISTS test_id_idx ON test_ids (id);

CREATE OR REPLACE FUNCTION refresh_test_ids()
    RETURNS TRIGGER LANGUAGE plpgsql
AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY test_ids;
    RETURN NULL;
END $$;

CREATE TRIGGER on_test_change
    AFTER INSERT OR UPDATE OR DELETE OR TRUNCATE
    ON test_entity
    FOR EACH STATEMENT
EXECUTE PROCEDURE refresh_test_ids();
```

In order to solve this task, you have to implement each Dockerfile of both services.
It is important that each service is served on port `8080` - docker-compose will 
remap them to avoid conflicts.

If you have merged the starter branch, you don't have to worry about database access.
Everything is already configured in the `application.yml`:

```yaml
 jpa:
    database: postgresql
    open-in-view: false
  datasource:
    platform: postgres
    url: jdbc:postgresql://patient-database:5432/patientdb
    username: cgmuser
    password: cgmpassword
```

**Hint 1: Entities** How to create an entity

```java
@Entity
@Table(name = "my_entity", schema = "public")
public class MyEntity {
    @Id
    @Column(name = "id")
    private String id;

    @Column(name = "column_name")
    private String someProp;
}
```

**Hint 2: Repositories** How to create a Repository

```java
@Repository
public interface SomeRepo extends 
    JpaRepository<MyEntity, String>, 
    JpaSpecificationExecutor<MyEntity> 
{ ... }
```
 
### 2.3 Deploy services into k8s cluster  

Now it is time for Kubernetes 😎. Get ready by executing the following scripts:
```bash
./scripts/setupKubernetes
./scripts/setupMinikube
```
After the installation was successful, you have to create your minikube cluster by running the following command:
```bash
minikube start --driver=virtualbox
```
By default, not all required features, that we need for our workshop, are enabled. To change this, run the following:
```bash
minikube addons enable default-storageclass
minikube addons enable ingress
```

Since we are working in a local environment and don't want to rely on an external docker registry, 
you have need to access the minikube's docker environment. This is done by running:
```bash
eval $(minikube docker-env)
``` 
You will notice new docker containers, if you run ``docker ps`` and probably recognize that your docker 
containers from task 2 are gone. Change this by building them again:
```bash
cd patient-service && docker build -t patient-service:1.0.0
cd insurance-service && docker build -t insurance-service:1.0.0
```

Before you start with the upcoming task, merge the branch ``task3/starter`` into your current one. 
It will add a new directory ``k8s``, which you will use to create your kubernetes files.

The task is as simple as that:
1) Complete all the stubbed k8s files and deploy them
2) Your ingress has to be reachable on `kubernetes-workshop.info`
3) Services should be of type cluster ip
4) Sensitive configurations like passwords should live inside a secret
5) All files that already exist as stubs have to be used and no additional ones should be created
6) Verify your result by running the integration tests

**Hint 1: Hostname** In order to get access to this hostname, you have to update your hosts file as follows:
```
# Get your cluster ip:
minikube ip

# Add this to your hosts file (/etc/hosts)
<the ip> kubernetes-workshop.info
```

**Hint 2: path resolution** You will face the issue, that subpaths are not automatically resolved by your ingress controller.
To solve this issue, you have to use the redirect feature of ingress-nginx like in the following example:
```
# Ingress.yaml
...
    annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /$2

...
    - path: /some-path-to-service(/|$)(.*)
...
```
This will append any subpath to your service call.

**Hint 3: Secrets** The value of a secret have to be encoded in base64. The following commands will en- and decode values:
```bash
# Encode in Base64
echo -n 'username' | base64
# Decode from Base64
echo -n 'dXNlcm5hbWU=' | base64 --decode
```

**Hint 4: DNS resolution** Just like in docker-compose, you can access different nodes in the cluster by using the 
service label. Consequently, the following use case demonstrated how to access a database:

```
# DBService.yaml
....
    selector:
        ports:
          - port: 5432
        app: patient-database
...

URL to access inside the cluster: patient-database:5432
```

**Hint 5: Configmap**
Inside the k8s/patient-db directory, you will find a Configmap. This is used to initialize your
postgresql database. Use the secret for database name, username and password.


The following commands will support you during development:
```bash
# Get all deployments
kubectl get deployments
# Inspect a given deployment
kubectl describe deployment <deployment-name>

# Get all pods
kubectl get pods
# Inspect a given pod
kubectl describe pod <pod-name>
# View logs of pod
kubectl logs -f <pod-name>

# Get all persistent volume claims
kubectl get pvc
# Inspect a given persistent volume claim
kubectl describe <pvc-name>

# Get all ingress
kubectl get ingress
# Inspect an ingress
kubectl describe ingress <ingress-name>

# Get all Services
kubectl get services
# Inspect a given service
kubectl describe service <service-name>

# Get all secrets
kubectl get secrets
# Inspect a secret
kubectl describe secret <secret-name>
# 

# Apply all k8s files of a given directory
kubectl apply -f <dir-name>

# Remove all k8s files from minikube located in a host dir
kubectl delete -f <dir-name>
```

### 2.3.1 Ingress

Use an ingress controller to route traffic from outside towards one of your services.
An ingress definition file looks as follows:
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
      # $2 takes everything behind the first path  
      nginx.ingress.kubernetes.io/rewrite-target: /$2
  # some more metadata
spec:
  rules:
    - host: your-domain.com
      http:
        paths:
          # The wildcard fowards any additional sub paths to the service
          - path: /example-route(/|$)(.*)
            backend:
              serviceName: example-service
              servicePort: 1234
          # More routes
```

Check the official [docs](https://kubernetes.io/docs/concepts/services-networking/ingress/) for more information.

### 2.3.2 Services

A service takes requests from inside the cluster and from an ingress controller
and dispatches it into a given replicaset that is using this service. For this 
purpose the following service file is required:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: some-service-name
  namespace: default
  # some additional metadata
spec:
  ports:
    - port: 1234
      protocol: TCP
  selector:
    app: some-selector
```

The selector is necessary for connecting deployments to this service and to access it; It allows 
to route requests to this label. Check the official [docs](https://kubernetes.io/docs/concepts/services-networking/service/) for more information.

### 2.3.3 Deployment

A deployment creates a set of pods with some additional features that might be of interest in
later iterations of this workshop. For this workshop the following aspects are relevant:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: some-deployment-name
  namespace: default
  # some more metadata
spec:
  replicas: 1
  selector:
    matchLabels:
      app: some-selector
  template:
    metadata:
      labels:
        app: some-selector
    spec:
      volumes:
        - name: some-configmap-volume
          configMap:
            name: some-configmap-name
        - name: some-volume
          persistentVolumeClaim:
            claimName: some-claim-name
      containers:
        - name: some-container-name
          imagePullPolicy: IfNotPresent
          image: insurance-service:1.0.0 # the docker image name
          ports:
            - containerPort: 8080
              name: insurance-http
          env:
            - name: SOME_ENV
              valueFrom:
                secretKeyRef:
                  name: some-secret-name
                  key: some-key-name
          volumeMounts:
            - mountPath: /path/in/container
              name: name-of-volume
```
 
Check the official [docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) for more information.

### 2.3.4 Configmap

A configmap can store files and configurations that have to be mounted into a given container:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: default
  name: some-configmap-name
data:
  some.sql: |
    SOME SQL STATEMENT;
```

Check the official [docs](https://kubernetes.io/docs/concepts/configuration/configmap/) for more information.

### 2.3.5 Secrets

A secret can store sensitive data which is mounted into the container just like a configmap.
Its values are encoded in base64 and one usually uses a tool to manage them (KMS). It's 
structure looks similar to a configmap:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: some-secret-name
type: Opaque
data:
  someKey: cGF0aWVudGRi
``` 

Check the official [docs](https://kubernetes.io/docs/concepts/configuration/secret/) for more information.

### 2.3.6 Persistent Volume Claim

A PersistentVolumeClaim (PVC) is a request for storage by a user. Its structure looks as follows:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: some-pvc-name
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
``` 

Check the official [docs](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) for more information.

## 3 Integration Tests

This module is already implemented and doesn't require additional changes. 
In short, it proves that your artifacts are working.

Run it with:
```bash
cd integration-tests && mvn clean verify
```

If you need to configure your tests (e.g. ports have changed or routes), 
then have a look into the following [file](./integration-tests/src/test/resources/config.yml).
It lets you change the host, port, baseUrl and health uri. These are required to run 
the integration tests successfully. 

## 4 Feedback

Feel free to leave feedback in a Github Issue. 

## 5 Upcoming

1) Service Mesh
2) Monitoring
3) Get it into a cloud
4) Kubernetes Package Manager
