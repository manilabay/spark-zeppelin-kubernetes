# spark-zeppelin-kubernetes

For this PoC, I will use:
* Google Container Engine. It could be any cloud with Kubernetes cluster Capability installed.
* Kubernetes
* Apache Spark
* Apache Zeppelin

So, the idea is to launch Spark and Zeppelin over Kubernetes over Google Cloud

First step lets create a Container Engine cluster with storage-full scopes.

```console
$ gcloud container clusters create spark --scopes storage-full
--machine-type n1-standard-4
```
Note: I'm using n1-standard-4 (larger than the default node size) to demonstrate some features of Horizontal Pod Autoscaling. However, Spark works fine on the default node size of n1-standard-1.

Now lets launch Spark on Kubernetes using the configuration files in the Kubernetes GitHub repo:

```console
$ git clone https://github.com/kubernetes/kubernetes.git
$ kubectl create -f kubernetes/examples/spark
```

‘kubernetes/examples/spark’ is a directory with all of the Spark YAML files to create the Kubernetes-Spark cluster.

The pods (especially Apache Zeppelin) are somewhat large, so may take some time for Docker to pull the images. Once everything is running, you should see something similar to the following:

```console
$ kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
spark-master-controller-v4v4y   1/1       Running   0          21h
spark-worker-controller-7phix   1/1       Running   0          21h
spark-worker-controller-hq9l9   1/1       Running   0          21h
spark-worker-controller-vwei5   1/1       Running   0          21h
zeppelin-controller-t1njl       1/1       Running   0          21h
```

You will see now Kubernetes is running one instance of Zeppelin, one Spark master and three Spark workers.
