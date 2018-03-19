# spark-zeppelin-kubernetes

For this PoC, I will use:
* Google Container Engine (GCE). It could be any cloud with Kubernetes cluster Capability installed.
* Google Cloud Storage (GCS)
* Kubernetes
* Apache Spark
* Apache Zeppelin

So, the idea is to launch Spark and Zeppelin over Kubernetes over Google cloud

## References

* [kubectl CheatSheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
* [PySpark CheatSheet](https://s3.amazonaws.com/assets.datacamp.com/blog_assets/PySpark_Cheat_Sheet_Python.pdf)
* [Spark Perf tests](https://github.com/databricks/spark-perf)

## GCloud Deployment Manager

This guide assume you have already intalled and setup locally your gcloud environment. If not please go link below and setup it.

[Google Cloud Deployment Manager](https://cloud.google.com/deployment-manager/docs/step-by-step-guide/installation-and-setup) (aka gcloud)

Additionally will useful if you install gcloud kubectl component

```console
gcloud components install kubectl
```

## Setup Spark cluster over Kubernetes

First step lets create a Container Engine cluster with storage-full scopes.

```console
$ gcloud container clusters create spark --scopes storage-full --machine-type n1-standard-4
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

## Set up the Secure Proxy to Zeppelin
Next, let's set up a secure proxy from your local machine to Zeppelin, to access the Zeppelin instance running in the cluster from your local machine. (Note: You’ll need to change this command to the actual Zeppelin pod name, specially the last 5 letters, you can check in above output)
```console
$ kubectl port-forward zeppelin-controller-t1njl 8080:8080
```

This will allow you to use Zeppelin safely.
Now, you can browse http://localhost:8080/ and see the Zeppelin intro page.


## Setup the PoC Data part

As an example, I'm going to show you how to build a simple movie recommendation model. It is based on the code on the Spark website, modified to put interest in our Kubernetes PoC.

### Import

Click “Import note,” give it an arbitrary name (e.g. “Movies”), and click “Add from URL.” and enter:
https://gist.githubusercontent.com/zmerlynn/875fed0f587d12b08ec9/raw/6eac83e99caf712482a4937800b17bbd2e7b33c4/movies.json

Click “Import Note.” This will give you a ready-made Zeppelin note for this demo. You should now have a “Movies” notebook, or what name you put before. Click that note.

You can now click the Play button and you’ll create a new, in-memory movie recommendation model! In the Spark application model, Zeppelin acts as a Spark Driver Program, interacting with the Spark cluster master to get its work done. In this case, the driver program that’s running in the Zeppelin pod fetches the data and sends it to the Spark master, which farms it out to the workers, which crunch out a movie recommendation model using the code from the driver. With a larger data set in Google Cloud Storage (GCS), it would be easy to pull the data from GCS as well.

## Working with Google Cloud Storage (Optional)
For this demo, we’ll be using Google Cloud Storage, which will let us store our model data beyond the life of a single pod. Spark for Kubernetes is built with the Google Cloud Storage connector built-in. As long as you can access your data from a virtual machine in the Google Container Engine project where your Kubernetes nodes are running, you can access your data with the GCS connector on the Spark image.

If you want, you can change the variables at the top of the note so that the example will actually save and restore a model for the movie recommendation engine — just point those variables at a GCS bucket that you have access to. If you want to create a GCS bucket, you can do something like this on the command line:
```console
$ gsutil mb gs://my-spark-movie-models
```
You’ll need to change this URI to something that is unique for you. This will create a bucket that you can use in the example above.

Note: Computing the model and saving it is much slower than computing the model and throwing it away. This is expected. However, if you plan to reuse a model, it’s faster to compute the model and save it and then restore it each time you want to use it, rather than throw away and recompute the model each time.

## Using Horizontal Pod Autoscaling with Spark (Optional)
Spark is elastic to workers coming and going, so now we will use Kubernetes Horizontal Pod Autoscaling feature to scale-out the Spark worker pool automatically, setting a target CPU threshold for the workers and a minimum/maximum pool size. This obviates the need for having to configure the number of worker replicas manually.

Create the Autoscaler like this (note: if you didn’t change the machine type for the cluster, you probably want to limit the --max to something smaller):  
```console
$ kubectl autoscale --min=1 --cpu-percent=80 --max=10 rc/spark-worker-controller
```
To see the full effect of autoscaling, wait for the replication controller to settle back to one replica. Use ‘kubectl get rc’ and wait for the “replicas” column on spark-worker-controller to fall back to 1.

The workload we ran before ran too quickly to be terribly interesting for HPA. To change the workload to actually run long enough to see autoscaling become active, change the “rank = 100” line in the code to “rank = 200.” After you hit play, the Spark worker pool should rapidly increase to 20 pods. It will take up to 5 minutes after the job completes before the worker pool falls back down to one replica.

Conclusion
In this PoC, I showed you:
* how to run Spark and Zeppelin on Kubernetes
* how to use Google Cloud Storage to store your Spark model
* how to use Horizontal Pod Autoscaling to dynamically size your Spark worker pool

## Delete the PoC cluster
Once, you learn it, if you want to delete the entire cluster you can do it with next command. Of course you could modify to your custom requirements for the yaml files used and follow and running same commands in this guide to build your environment.

```console
$ gcloud container clusters delete spark
```
