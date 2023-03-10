With Iter8, you can launch experiments declaratively and imperatively. The focus of this article is to showcase the former by describing how you can configure Iter8 to automatically test an HTTP application whenever you update it and ensure specific endpoints meet latency and error-related SLOs. Later on, we will also describe how to modify the configuration to do the same for gRPC endpoints.

# Automatic performance testing for multiple HTTP endpoints

### Download Iter8 CLI

```bash
brew tap iter8-tools/iter8
brew install iter8@0.13
```

The Iter8 CLI provides the commands needed to see experiment reports. See [here](https://iter8.tools/0.11/getting-started/install/) for alternate methods of installation.

### Setup Kubernetes cluster with ArgoCD

As mentioned previously, AutoX is built on top of ArgoCD, so we will also need to install it.

A basic install of Argo CD can be done as follows:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

See [here](https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd) for more information.

### Deploy HTTP application

Now, we will create an `httpbin` deployment and service. The `httpbin` deployment will become the trigger object, meaning AutoX will respond to changes to this Kubernetes object.

```bash
kubectl create deployment httpbin --image=kennethreitz/httpbin --port=80
kubectl expose deployment httpbin --port=80
```

### Apply version label

Next, we will assign `httpbin` deployment the `app.kubernetes.io/version` label (version label). AutoX will only launch experiments if the trigger object has a version label. AutoX will relaunch experiments whenever this version label is modified.

```bash
kubectl label deployment httpbin app.kubernetes.io/version=1.0.0
```

### Configure AutoX

Next, we will configure and install the AutoX controller.

```bash
helm install autox autox --repo https://iter8-tools.github.io/hub/ --version 0.1.6 \
--set 'groups.httpbin.trigger.name=httpbin' \
--set 'groups.httpbin.trigger.namespace=default' \
--set 'groups.httpbin.trigger.group=apps' \
--set 'groups.httpbin.trigger.version=v1' \
--set 'groups.httpbin.trigger.resource=deployments' \
--set 'groups.httpbin.specs.iter8.name=iter8' \
--set 'groups.httpbin.specs.iter8.values.tasks={ready,http,assess}' \
--set 'groups.httpbin.specs.iter8.values.ready.deploy=httpbin' \
--set 'groups.httpbin.specs.iter8.values.ready.service=httpbin' \
--set 'groups.httpbin.specs.iter8.values.ready.timeout=60s' \
--set 'groups.httpbin.specs.iter8.values.http.numRequests=200' \
--set 'groups.httpbin.specs.iter8.values.http.endpoints.get.url=http://httpbin.default/get' \
--set 'groups.httpbin.specs.iter8.values.http.endpoints.getAnything.url=http://httpbin.default/anything' \
--set 'groups.httpbin.specs.iter8.values.http.endpoints.post.url=http://httpbin.default/post' \
--set 'groups.httpbin.specs.iter8.values.http.endpoints.post.payloadStr=hello' \
--set 'groups.httpbin.specs.iter8.values.assess.SLOs.upper.http/get/error-count=0' \
--set 'groups.httpbin.specs.iter8.values.assess.SLOs.upper.http/get/latency-mean=50' \
--set 'groups.httpbin.specs.iter8.values.assess.SLOs.upper.http/getAnything/error-count=0' \
--set 'groups.httpbin.specs.iter8.values.assess.SLOs.upper.http/getAnything/latency-mean=75' \
--set 'groups.httpbin.specs.iter8.values.assess.SLOs.upper.http/post/error-count=0' \
--set 'groups.httpbin.specs.iter8.values.assess.SLOs.upper.http/post/latency-mean=100' \
--set 'groups.httpbin.specs.iter8.version=0.13.0' \
--set 'groups.httpbin.specs.iter8.values.runner=job'
```

The AutoX controller configuration is composed of a *trigger object definition* and a set of *experiment specifications*. In this case, the trigger object is the `httpbin` deployment and there is only one experiment, an HTTP performance test with SLO validation associated with this trigger.

To go into more detail, the configuration is composed of a set of *groups*, and each group is composed of a trigger object definition and a set of *experiment specifications*. This enables AutoX to manage one or more trigger objects, each associated with one or more experiments. In this tutorial, there is only one group named `httpbin` (`groups.httpbin...`), and within that group, there is the trigger object definition (`groups.httpbin.trigger...`) and a single experiment spec named `iter8` (`groups.httpbin.specs.iter8...`). 

The trigger object definition is a combination of the name, namespace, and the group-version-resource (GVR) metadata of the trigger object, in this case `httpbin`, `default`, and GVR `apps`, `deployments`, and `v1`, respectively. 

The experiment is an HTTP SLO validation test on the `httpbin` service that is described in greater detail [here](https://iter8.tools/0.13/getting-started/your-first-experiment/). This Iter8 experiment is composed of three tasks, `ready`, `http`, and `assess`. The `ready` task will ensure that the `httpbin` deployment and service are running. The `http` task will generate load for the specified endpoints, `get`, `getAnything`, and `post`, and collect latency and error-related metrics. Lastly, the `assess` task will ensure that the error count is 0 for each endpoint and the mean latency is less than 50, 75, and 100 milliseconds for the `get`, `getAnything`, and `post`, respectively. In addition, the runner is set to job as this will be a [single-loop experiment](https://iter8.tools/0.11/getting-started/concepts/#iter8-experiment).

### Observe experiment

After starting AutoX, the HTTP SLO validation test should quickly follow. You can now use the Iter8 CLI in order to check the status and see the results of the test. 

The following command allows you to check the status of the test. Note that you need to specify an experiment group via the `-g` option. The *experiment group* for experiments started by AutoX is in the form `autox-<group name>-<experiment spec name>` so in this case, it would be `autox-httpbin-iter8`.

```bash
iter8 k assert -c completed -c nofailure -c slos -g autox-httpbin-iter8
```

We can see in the sample output that the test has completed, there were no failures, and all SLOs and conditions were satisfied.
```
INFO[2023-01-11 14:43:45] inited Helm config          
INFO[2023-01-11 14:43:45] experiment completed
INFO[2023-01-11 14:43:45] experiment has no failure                    
INFO[2023-01-11 14:43:45] SLOs are satisfied                           
INFO[2023-01-11 14:43:45] all conditions were satisfied  
```

***

The following command allows you to see the results as a text report.

```bash
iter8 k report -g autox-httpbin-iter8
```

You can also produce an HTML report that you can view in the browser.

```bash
iter8 k report -g autox-httpbin-iter8 -o html > report.html
```

The HTML report will look similar to the following:
![HTML report](images/htmlreport.png)

### Continuous and automated experimentation

Now that AutoX is watching the `httpbin` deployment, releasing a new version will relaunch the HTTP SLO validation test. The update must be accompanied by a version label change change; otherwise, AutoX will not do anything.

For simplicity, we will simply change the version label in order to relaunch the HTTP SLO validation test. In the real world, a new version would typically involve a change to the deployment spec (e.g., the container image) and this change should be accompanied by a change to the version label.

```bash
kubectl label deployment httpbin app.kubernetes.io/version=2.0.0 --overwrite
```

### Observe new experiment

Check if a new experiment has been launched. Refer to [Observe experiment](#observe-experiment) for the necessary commands.

If we were to continue to update the deployment (and change its version label), then AutoX would relaunch the experiment for each such change.

***

# Automatic performance testing for multiple gRPC endpoints

First, we will create an example gRPC server. 

We will use the [route guide example](https://github.com/grpc/grpc-go/tree/master/examples/route_guide) provided by gRPC.

```bash
kubectl create deployment routeguide --image=golang --port=50051 \
-- bash -c "git clone -b v1.52.0 --depth 1 https://github.com/grpc/grpc-go; cd grpc-go/examples/route_guide; go run server/server.go"
kubectl expose deployment routeguide --port=50051
```

### Configure AutoX

```bash
helm install autox autox --repo https://iter8-tools.github.io/hub/ --version 0.1.6 \
--set 'groups.grpc-example.trigger.name=grpc-example' \
--set 'groups.grpc-example.trigger.namespace=default' \
--set 'groups.grpc-example.trigger.group=apps' \
--set 'groups.grpc-example.trigger.version=v1' \
--set 'groups.grpc-example.trigger.resource=deployments' \
--set 'groups.grpc-example.specs.iter8.name=iter8' \
--set 'groups.grpc-example.specs.iter8.values.tasks={ready,http,assess}' \
--set 'groups.grpc-example.specs.iter8.values.ready.deploy=grpc-example' \
--set 'groups.grpc-example.specs.iter8.values.ready.service=grpc-example' \
--set 'groups.grpc-example.specs.iter8.values.ready.timeout=60s' \
--set 'groups.grpc-example.specs.iter8.values.grpc.host=grpc-example.default:81' \
--set 'groups.grpc-example.specs.iter8.values.grpc.endpoints.getFeature.call=routeguide.RouteGuide.GetFeature' \
--set 'groups.grpc-example.specs.iter8.values.grpc.endpoints.getFeature.dataURL=...' \
--set 'groups.grpc-example.specs.iter8.values.grpc.endpoints.listFeatures.call=routeguide.RouteGuide.ListFeatures' \
--set 'groups.grpc-example.specs.iter8.values.grpc.endpoints.listFeatures.dataURL=...' \
--set 'groups.grpc-example.specs.iter8.values.grpc.protoURL=https://raw.githubusercontent.com/grpc/grpc-go/master/examples/route_guide/routeguide/route_guide.proto' \
--set 'groups.grpc-example.specs.iter8.values.assess.SLOs.upper.grpc/getFeature/error-rate=0' \
--set 'groups.grpc-example.specs.iter8.values.assess.SLOs.upper.grpc/getFeature/latency-mean=100' \
--set 'groups.grpc-example.specs.iter8.values.assess.SLOs.upper.grpc/listFeatures/error-rate=0' \
--set 'groups.grpc-example.specs.iter8.values.assess.SLOs.upper.grpc/listFeatures/latency-mean=100' \
--set 'groups.grpc-example.specs.iter8.version=0.13.0' \
--set 'groups.grpc-example.specs.iter8.values.runner=job'
```

<!-- ### Additional methods of configuration

The `http` task has additional [parameters](https://iter8.tools/0.13/user-guide/tasks/http/#parameters) that the user can set.

For example, the `qps` (queries-per-second) has a default value of `8`. There are multiple ways of utilizing this parameter in the context of multiple endpoints.

In the following example, the default value is used.

```bash
--set 'groups.httpbin.specs.iter8.values.http.endpoints.getit.url=http://httpbin.default/get' \
--set 'groups.httpbin.specs.iter8.values.http.endpoints.post.url=http://httpbin.default/post' \
--set 'groups.httpbin.specs.iter8.values.http.endpoints.post.payloadStr=hello' \
```

The following example, the default value is overridden to `10`. Both endpoints will be queried at 10 queries-per-second.

```bash
--set 'groups.httpbin.specs.iter8.values.http.qps=10' \
--set 'groups.httpbin.specs.iter8.values.http.endpoints.getit.url=http://httpbin.default/get' \
--set 'groups.httpbin.specs.iter8.values.http.endpoints.post.url=http://httpbin.default/post' \
--set 'groups.httpbin.specs.iter8.values.http.endpoints.post.payloadStr=hello' \
```

The following example, the default value is overridden to `10`, but one of `getit` endpoint has overridden that value to `15`. `getit` will be queried at 15 queries-per-second whereas `post` will be queried at 15 queries-per-second whereas .

```bash
--set 'groups.httpbin.specs.iter8.values.http.qps=10' \
--set 'groups.httpbin.specs.iter8.values.http.endpoints.getit.url=http://httpbin.default/get' \
--set 'groups.httpbin.specs.iter8.values.http.endpoints.getit.qps=15' \
--set 'groups.httpbin.specs.iter8.values.http.endpoints.post.url=http://httpbin.default/post' \
--set 'groups.httpbin.specs.iter8.values.http.endpoints.post.payloadStr=hello' \
``` -->
