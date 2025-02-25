<img src="/images/flow.png" width="250">

### Create a Kubernetes cluster

`kind create cluster`

### Add the bitnami helm chart

`helm repo add bitnami https://charts.bitnami.com/bitnami`

`helm repo update`

### Install Postgres and test

#### Create a Persistant Volume

```
kubectl apply -f- <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgresql-pv
  labels:
    type: local
spec:
  storageClassName: standard
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
EOF
```

#### Install Postgres

`helm install psql-test bitnami/postgresql --set volumePermissions.enabled=true`

#### Get the Postgres password

`export POSTGRES_PASSWORD=$(kubectl get secret --namespace default psql-test-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)`

#### Test connectivity

```
kubectl run psql-test-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:16.4.0-debian-12-r0 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host psql-test-postgresql -U postgres -d postgres -p 5432
```

#### (optional) Install `libpq`

`brew install libpq`

```
echo 'export PATH="/opt/homebrew/opt/libpq/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

#### (optional) Test connectivity

`kubectl port-forward --namespace default svc/psql-test-postgresql 5432:5432`

`PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432`

### Deploy Hasura and test

#### Deploy Hasura

Update the Postgres password under `HASURA_GRAPHQL_DATABASE_URL` in the YAML block below before running

```
kubectl apply -f- <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hasura
    hasuraService: custom
  name: hasura
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hasura
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: hasura
    spec:
      containers:
      - image: hasura/graphql-engine:v2.42.0
        imagePullPolicy: IfNotPresent
        name: hasura
        env:
        - name: HASURA_GRAPHQL_DATABASE_URL
          value: postgres://postgres:PASSWORD@psql-test-postgresql:5432/postgres
        ## enable the console served by server
        - name: HASURA_GRAPHQL_ENABLE_CONSOLE
          value: "true"
        ## enable debugging mode. It is recommended to disable this in production
        - name: HASURA_GRAPHQL_DEV_MODE
          value: "true"
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /healthz
            port: http
        readinessProbe:
          httpGet:
            path: /healthz
            port: http
        resources: {}
EOF
```
```
kubectl apply -f- <<EOF
apiVersion: v1
kind: Service
metadata:
  labels:
    app: hasura
  name: hasura
  namespace: default
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  selector:
    app: hasura
  type: LoadBalancer
EOF
```

#### Test

`kubectl port-forward --namespace default svc/hasura 8000:80`

http://localhost:8000/console

Add Data to Hasura: [tutorial](https://www.youtube.com/watch?v=ZGKQ0U18USU&t=5s&ab_channel=Hasura)

Get all posts:

```
curl 'http://localhost:8000/v1/graphql' \
  -X POST \
  -H 'content-type: application/json' \
  --data '{
    "query": "{ posts { created_at id title updated_at } }"
  }' | jq
  ```

Add the Starlink GraphQL remote schema and test:

Navigate to http://localhost:8000/console/remote-schemas/manage/schemas

Add `https://spacex-production.up.railway.app/`

```
curl 'http://localhost:8000/v1/graphql' \
  -X POST \
  -H 'content-type: application/json' \
  --data '{
    "query": "{ launches { launch_date_local mission_name rocket { rocket_name } } }"
  }' | jq
```

### Deploy kgateway

`kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml`

`helm install --create-namespace --namespace kgateway-system --version v2.0.0-main kgateway oci://ghcr.io/kgateway-dev/charts/kgateway`

### Create a Gateway and test

#### Create a Gateway

```
kubectl apply -f- <<EOF
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: graphql-gw
  namespace: default
spec:
  gatewayClassName: kgateway
  listeners:
  - protocol: HTTP
    port: 8080
    name: http
    allowedRoutes:
      namespaces:
        from: All
EOF
```
```
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: graphql-route
  namespace: default
spec:
  parentRefs:
    - name: graphql-gw
      namespace: default
  rules:
    - matches:
      - path:
          type: PathPrefix
          value: /
      backendRefs:
      - name: hasura
        port: 80
EOF
```

#### Test connectivity through the gateway

`kubectl port-forward deployment/graphql-gw 8080:8080`

http://localhost:8080/console

Get all posts (through kgateway):

```
curl 'http://localhost:8080/v1/graphql' \
  -X POST \
  -H 'content-type: application/json' \
  --data '{
    "query": "{ posts { created_at id title updated_at } }"
  }' | jq
  ```

Test the Starlink remote GraphQL endpoint:

```curl 'http://localhost:8000/v1/graphql' \
  -X POST \
  -H 'content-type: application/json' \
  --data '{
    "query": "{ launches { launch_date_local mission_name rocket { rocket_name } } }"
  }' | jq
```

### Cleanup

`kind delete cluster`

### Links

https://phoenixnap.com/kb/postgresql-kubernetes

https://hasura.io/docs/2.0/deployment/deployment-guides/kubernetes/

https://kgateway.dev/docs/quickstart/