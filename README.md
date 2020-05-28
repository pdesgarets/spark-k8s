# How to use Spark on Kubernetes

Following applies for Python jobs. Should be ~the same for Scala/Java.

You need :

- a Kubernetes cluster on which you can create serviceaccounts, clusterrole, rolebinding, pods and services, and the command line tool `kubectl` to do so
- a python job, as a Python file
- a base Docker Image with Spark, for example `my.registry.io/repository/base-spark:latest`
- [jq](https://stedolan.github.io/jq/) to easily manipulate data from kubectl

This simple tutorial assumes that your docker images are on public repositories, and your job fits in a single python file without any dependencies.

## How To create a base image

>(If you want to skip this part, you can use `pdesgarets/spark-py:latest` for Spark for Python v2.4.5)

As written in the [docs](https://spark.apache.org/docs/latest/running-on-kubernetes.html), you can use the [docker-image-tool.sh](https://github.com/apache/spark/blob/master/bin/docker-image-tool.sh).

The fastest way is to use docker-in-docker build, using a Docker image with Spark installed. For instance with `bitnami/spark:2.4.5` (replace `my.registry.io/repository/base-spark`) :

```
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock -u 0 --privileged bitnami/spark:2.4.5 bin/docker-image-tool.sh -r docker.io/my.registry.io/repository/base-spark -t latest -p resource-managers/kubernetes/docker/src/main/dockerfiles/spark/bindings/python/Dockerfile build
```

## How To

1. We want to run our job in a dedicated Namespace called `spark`, using a dedicated ServiceAccount called `spark-na`. So we create all of this, with the permissions : `kubectl create -f rbac.yaml`
2. We will use a dedicated docker image, containing Spark *and* our job. So we write a `Dockerfile` (replace `my.registry...` with your base image) :

	```Dockerfile
	FROM my.registry.io/repository/base-spark:latest

	COPY script.py /opt/custom_jobs/script.py
	```

	and we build the new image (replace `my.registry...` with the name of your new custom image):

	```bash
	export JOB_IMAGE_NAME="my.registry.io/repository/my-spark-job:latest"
	docker build -t $JOB_IMAGE_NAME .
	docker push $JOB_IMAGE_NAME
	```

3. To launch the job, we need to connect to the cluster. To connect, we need
	1. a hostname
	2. a CertificateAuthority certificate
	3. a token

	#### Hostname
	`kubectl cluster-info` gives you "Kubernetes master is running at https://the.hostname.com".

	Put `the.hostname.com` in a variable : `export KUBE_API_HOST=the.hostname.com`

	#### CertificateAuthority and token

	We will use the token granted to the service account `spark-sa` we created earlier :

	```
	export SPARK_TOKEN=$(kubectl -n spark get secret $(kubectl -n spark get serviceaccount spark-sa -o json|jq --raw-output ".secrets[0].name") -o json |jq -r ".data.\"token\"" |base64 --decode)
	```

	And the matching CA:

	```
	kubectl -n spark get secret $(kubectl -n spark get serviceaccount spark-sa -o json|jq --raw-output ".secrets[0].name") -o json |jq -r ".data.\"ca.crt\"" |base64 --decode > ca.crt
	```
	The CA should begin with `-----BEGIN CERTIFICATE-----` and end with `-----END CERTIFICATE-----`


4. We got everything, we can submit the job. It can be done from anywhere as long as you have the Kubernetes API Hostname, the CertificateAuthority and a token. Here we will do it in the container we created earlier, to avoid installing Spark on our PC :

	```bash
	docker run --rm -it -v $PWD/ca.crt:/tmp/ca.crt $JOB_IMAGE_NAME /opt/spark/bin/spark-submit \
	  --master k8s://https://$KUBE_API_HOST:443 \
	  --deploy-mode cluster \
	  --name my-job \
	  --conf spark.kubernetes.namespace=spark \
	  --conf spark.executor.instances=2 \
	  --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark-sa \
	  --conf spark.kubernetes.container.image=$JOB_IMAGE_NAME \
	  --conf spark.kubernetes.authenticate.submission.caCertFile=/tmp/ca.crt \
	  --conf spark.kubernetes.authenticate.submission.oauthToken=$SPARK_TOKEN \
	  --executor-cores 1 --executor-memory 512M \
	  local:///opt/custom_jobs/script.py
	```

	What happens :

	1. It creates a pod with a Spark Driver
	2. It creates a Service so that you can get a Web UI to the driver
	3. The driver creates as many pods as the `executor.instances` setting
	4. The executor pods will run, then everything will shutdown once the job finished

	And voilà.



## What's next

There are many settings you can use, look at the docs. You will probably need to :

- use private Docker images : `spark.kubernetes.container.image.pullSecrets`
- inject secret env var : `spark.kubernetes.driver.secretKeyRef.[EnvName]` / `spark.kubernetes.executor.secretKeyRef.[EnvName]`
- use dependencies : ?
- use scala : use a different base image



## References


- https://cloud.google.com/solutions/spark-on-kubernetes-engine?hl=fr
- https://spark.apache.org/docs/latest/running-on-kubernetes.html
- https://medium.com/@tunguyen9889/how-to-run-spark-job-on-eks-cluster-54f73f90d0bc
