# Overview Broker
[![Build Status](https://travis-ci.org/mattmcneeney/overview-broker.svg?branch=master)](https://travis-ci.org/mattmcneeney/overview-broker)

![Open Service Broker API](images/openservicebrokerapi.png)

A simple service broker conforming to the [Open Service Broker API](https://github.com/openservicebrokerapi/servicebroker/)
specification that hosts a dashboard showing information on service instances
and bindings created by any platform the broker is registered with.

The broker supports has two plans: `simple` and `complex`. The `complex` plan
has a schema that is used to validate configuration parameters that are provided
when creating a service instance, updating a service instance or creating a
service binding.

> The [Open Service Broker API](https://www.openservicebrokerapi.org/) project
allows developers, ISVs, and SaaS vendors a single, simple, and elegant way to
deliver services to applications running within cloud native platforms such as
Cloud Foundry, OpenShift, and Kubernetes. The project includes individuals from
Fujitsu, Google, IBM, Pivotal, RedHat and SAP.

---

### Installation
```bash
npm install
```

### Running
```bash
npm start
```

### Tests

To run the test suite:
```bash
npm test
```

---

### Enabling persistence

The Overview Broker can provide persistence using the free
[keyvalue.xyz](https://keyvalue.xyz/) service. For the broker to start up
successfully in persistence mode, you must provide the following environment
variables:
* `ENABLE_PERSISTENCE`: `true`
* `KV_TOKEN`: the
[token](https://github.com/kvaas/docs/blob/master/REST%20API.md#post-newkey)
used to set and get the key value pair
* `KV_KEY_NAME`: the key name

You can create a new token (`KV_TOKEN`) using the following command. The token
will be included in the response before the name of your key:
```bash
$ curl -X POST https://api.keyvalue.xyz/new/KEY_NAME
https://api.keyvalue.xyz/TOKEN/KEY_NAME
```

You should then set `KV_TOKEN` to `TOKEN` and `KV_KEY_NAME` to `KEY_NAME`.
Note that the generated token can only be used to get and set the provided key
name.

---

### Platforms

#### Cloud Foundry

##### 1. Deploying the broker

* First you will need to deploy the broker as an application:
    ```bash
    cf push overview-broker --no-start
    ```
* Then enable persistence mode if required:
    ```bash
    cf set-env overview-broker ENABLE_PERSISTENCE true
    cf set-env overview-broker KV_TOKEN <TOKEN>
    cf set-env overview-broker KV_KEY_NAME <KEY_NAME>
    cf restage overview-broker
    ```
    You can also use an application manifest to deploy the broker as an
    application:
    ```bash
    wget https://raw.githubusercontent.com/mattmcneeney/overview-broker/master/examples/cloudfoundry/manifest.yaml
    cf push
    ```

##### 2. Registering the broker

* To register the broker to a space (does not require admin credentials), run:
    ```bash
    cf create-service-broker --space-scoped overview-broker admin password <url-of-deployed-broker>
    ```
* The overview broker dashboard should now be accessible at
`https://<url-of-deployed-broker>/dashboard`.


##### 3. Creating a service instance

* Now for the exciting part... it's time to create a new service instance:
    ```bash
    cf create-service overview-broker simple my-broker
    ```
    You can give your service a specific name in the dashboard by providing the
    `name` configuration parameter:
    ```bash
    cf create-service overview-broker simple my-broker -c '{ "name": "My Broker" }'
    ```
    Or, if you like all the colours of the rainbow...
    ```bash
    cf create-service overview-broker simple my-broker -c '{ "rainbow": true }'
    ```
* If you now head back to the dashboard, you should see your new service
instance information!

#### Kubernetes

##### 1. Deploying the broker

* If you want to deploy the broker on Kubernetes, you will first need to build an
image for the container. If you're using
[minikube](https://kubernetes.io/docs/getting-started-guides/minikube/), you
don't need to push the image to a registry. Instead you can just build the
image using the same Docker host as the minikube VM. To build a container using
the example [Dockerfile](/examples/kubernetes/Dockerfile) provided, run:
    ```bash
    wget https://raw.githubusercontent.com/mattmcneeney/overview-broker/master/examples/kubernetes/Dockerfile
    eval $(minikube docker-env)
    docker build -t overview-broker:v1 .
    ```
    If you already have Docker installed and running on your machine and want to
    use that daemon again, just run:
    ```bash
    eval $(minikube docker-env -u)
    ```
* Now it's time to create a deployment. A Kubernetes deployment manages a pod,
which we want to run our `overview-broker` image:
    ```bash
    kubectl run overview-broker --image=overview-broker:v1 --port=8080 --env="PORT=8080"
    ```
    If you wish to enable persistence, you will have to pass the required
    environmental variables when creating the deployment:
    ```bash
    kubectl run overview-broker --image=overview-broker:v1 --port=8080 --env="PORT=8080" --env="ENABLE_PERSISTENCE=true" --env="KV_KEY_NAME=<KEY_NAME>" --env="KV_TOKEN=<TOKEN>"
    ```
    You can check the deployment has succeeded by running `kubectl get deployments`
    and `kubectl get pods`.
* In order to access the Pod from outside of the Kubernetes virtual network, you
need to create a service. You can create a load balancer to do this easily:
    ```bash
    kubectl expose deployment overview-broker --type=LoadBalancer
    ```
* Check your service has been created successfully using `kubectl get services`.
    You should now be able to access the broker dashboard by running:
    ```bash
    minikube service overview-broker
    ```

##### 2. Registering the broker

To register the broker, you first need to install the Service Catalog using
Helm. _The instructions to do this are likely to change, so if anything below
breaks, then check out the [official guide](https://github.com/kubernetes-incubator/service-catalog/blob/master/docs/walkthrough.md)._

* If you don't already have Helm, then follow [these](https://github.com/kubernetes/helm/blob/master/docs/install.md)
installation instructions to install both Helm and Tiller. If you're on macOS,
the quickest way to do this is:
    ```bash
    brew install kubernetes-helm
    helm init
    ```
* The service catalog then needs to be installed with:
    ```bash
    git clone git@github.com:kubernetes-incubator/service-catalog.git ~/workspace/service-catalog
    helm install ~/workspace/service-catalog/charts/catalog --name catalog --namespace catalog
    SERVICE_CATALOG_SERVER=$(minikube service catalog-catalog-apiserver --url --namespace catalog | awk 'FNR==1')
    kubectl config set-cluster service-catalog --server=$SERVICE_CATALOG_SERVER
    kubectl config set-context service-catalog --cluster=service-catalog
    ```
    If everything installed successfully, then you should see the following:
    ```bash
    $ kubectl --context=service-catalog get brokers,serviceclasses,instances,bindings
    No resources found.
    ```
* Finally, you need to register a broker server with the catalog by creating
a new Broker resource. Download the provided manifest (`overview-broker.yaml`)
to do this, but be sure to edit the file to update the URL of the broker you
have deployed (use the IP address returned by
`minikube service overview-broker --url`):
    ```bash
    wget https://raw.githubusercontent.com/mattmcneeney/overview-broker/master/examples/kubernetes/overview-broker.yaml
    # Update <URL> to the output of 'minikube service overview-broker --url'
    ```
    Now create the resource with:
    ```bash
    kubectl --context=service-catalog create -f overview-broker.yaml
    ```
    Check the status of the broker with:
    ```bash
    kubectl --context=service-catalog get brokers ups-broker -o yaml
    ```
    If all has gone well, you should see a message saying `"Suffessully fetched
    catalog entries from broker"`.
    Your broker should also appear in the output from:
    ```bash
    kubectl --context=service-catalog get serviceclasses
    ```

##### 3. Creating a service instance

Now that we have the `overview-broker` ServiceClass within our cluster's service
catalog, we can provision a new Instance resource.

* Download the provided manfifest (`overview-broker-instance.yaml`) and create
a new instance with:
    ```bash
    wget https://raw.githubusercontent.com/mattmcneeney/overview-broker/master/examples/kubernetes/overview-broker-instance.yaml
    kubectl --context=service-catalog create -f overview-broker-instance.yaml
    ```
    Check everything has worked with:
    ```bash
    kubectl --context=service-catalog get instances overview-broker-instance -o yaml
    ```
    You should see a message saying `"The instance was provisioned successfully"`.
* If you now head back to the dashboard (run `minikube service overview-broker`),
you should see your new service instance information!
