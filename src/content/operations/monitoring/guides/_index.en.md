<article class="h-entry">

<header>

# Multi-Cluster monitoring with Prometheus and Submariner

</header>

<section data-field="subtitle" class="p-summary">Multi-Cluster deployments with Kubernetes are becoming very common, due to different reasons like redundancy, performance, scale, and more.</section>

<section data-field="body" class="e-content">

<section name="8b20" class="section section--body section--first section--last">

<div class="section-divider">

* * *

</div>

<div class="section-content">

<div class="section-inner sectionLayout--insetColumn">

### Multi-Cluster monitoring with Prometheus and Submariner

Multi-Cluster deployments with Kubernetes are becoming very common, due to different reasons like redundancy, performance, scale, and more.

Monitoring such an environment and its services is important. The most common monitoring system used in Kubernetes (at this moment) is the [Prometheus](https://prometheus.io/), which supplies metrics collection to a time series DB, alerting dashboards (using Grafana and more).

Submariner - as described on the Submariner website

> Submariner enables direct networking between Pods and Services in different Kubernetes clusters, either on-premises or in the cloud.

> To go through the next tutorial, it is recommended to get familiar with both _Prometheus_ and Submariner, but it is not mandatory as this tutorial shows a toy example in a local environment.

Submariner implements the [mcs-api](https://github.com/kubernetes-sigs/mcs-api) to allow communication between services and pods, on a multi-cluster setup. Combining with [Kube Federation](https://github.com/kubernetes-sigs/kubefed) (which allows federating resources such as namespace, deployments, and Custome Resources (CRDs) among clusters) — will allow full multi-cluster control for administrators. More about combining KubeFed and Submariner to achieve a really Multi-Cluster setup in my next post!

After this short prolog, let’s dive into our example on how setup a multi-cluster monitoring system. The following steps will show a toy example on how to connect two clusters with Submariner

Step 1 — Setup a local `Kind`environment:  
`[Kind](https://kind.sigs.k8s.io/)`is a tool depending on Docker to run multiple Kubernetes clusters locally.   
Hence you will need `Docker`, `Kind` and `kubectl`installed on your local machine.   
- For docker installation refer to [Get Docker](https://docs.docker.com/get-docker/).   
- For `Kind` installation refer to the [quick-start guide](https://kind.sigs.k8s.io/docs/user/quick-start/#installation).  
- For `kubectl` installation please refer to [Kubernetes Install](https://kubernetes.io/docs/tasks/tools/).  
I work with a Macbook, so I could install all the tools above on my local machine using Homebrew with one command.

    brew update && \ brew cask install docker && \brew install kind && \brew install kubectl

Step 2 — Spinning up some local clusters:  
We will use a boilerplate setup from _Submariner._  
Just download the _Submariner_ repository, and run `make clusters`   
This will spin up two local Kubernetes clusters.

<pre name="501c" id="501c" class="graf graf--pre graf-after--p">git clone [https://github.com/submariner-io/submariner](https://github.com/submariner-io/submariner)  
cd submariner  
make clusters</pre>

Please refer to [Submariner Quick-Start Guide](https://submariner.io/getting-started/quickstart/) for installations on different cloud providers.

Step 3 — Deploy `Prometheus` on the clusters:  
There are many ways for deploying _Prometheus_ on your Kubernetes cluster, I have used the kube-prometheus setup (some will recommend Helm which is great as well), which deploys prometheus-operator from this [link](https://github.com/prometheus-operator/kube-prometheus). But we are going to modify the quick start version to allow _Submariner_ metrics collection and monitoring as well.

Install [jsonnet](https://jsonnet.org/learning/getting_started.html), Again I’m using Mac, so I install using Homebrew

<pre name="2a4a" id="2a4a" class="graf graf--pre graf-after--p">brew install jsonnet</pre>

Download pre-defined configuration files and build the

<pre name="b872" id="b872" class="graf graf--pre graf-after--p">git cone [https://github.com/danibachar/submariner-cheatsheet](https://github.com/danibachar/submariner-cheatsheet.git)  
cd submariner-cheatsheet/prometheus/install  
jb init  
jb install github.com/prometheus-operator/kube-prometheus/jsonnet/kube-prometheus@release-0.8  
jb update  
sudo chmod +x ./build.sh  
./build.sh</pre>

After the script finishes running, you will notice several new directories under `submariner-cheatsheet/prometheus/install`. The one that we will find interesting is the `manifast` directory and its subdirectory `setup` under them there are all the yamls defining the _Prometheus_ CRDs and the operator.

Go back to the root directory where we downloaded the submariner repository into. And let's deploy _Prometheus_

<pre name="68f2" id="68f2" class="graf graf--pre graf-after--p">kubectl --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1 apply -f submariner-cheatsheet/prometheus/install/manifests/setup  
kubectl --kubeconfig submariner/output/kubeconfigs/kind-config-cluster2 apply -f submariner-cheatsheet/prometheus/install/manifests/setup  

kubectl --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1 apply -f submariner-cheatsheet/prometheus/install/manifests/  
kubectl --kubeconfig submariner/output/kubeconfigs/kind-config-cluster2 apply -f submariner-cheatsheet/prometheus/install/manifests/</pre>

Now you should see _Prometheus_ CRDS, services, and deployments starting to spin up. Run each of these commands to see the relevant Kubernetes resources.  
Note that _Prometheus_ is deployed in the monitoring namespace.  
Note that this example is using cluster1 config, change the kubeconfig flag to point to the cluster2 config to see the status there.

<pre name="59a3" id="59a3" class="graf graf--pre graf-after--p"># Note the new namespace `monitoring` has appearedkubectl --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1 get ns</pre>

<pre name="5b11" id="5b11" class="graf graf--pre graf-after--pre"># Will show all the new CRDs defined by Prometheus or any other setup (for example Submariner has its own CRDs)  
kubectl --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1 get crds</pre>

<pre name="ef4b" id="ef4b" class="graf graf--pre graf-after--pre"># Will show all the relevant service accounts, role and role bindings defined for Prometheus.  
kubectl --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1 get role, rolebindings, clusterrole,clusterrolebindings, serviceaccount -n monitoring</pre>

<pre name="1cfd" id="1cfd" class="graf graf--pre graf-after--pre"># Will show the service monitoring and prometheus setups  
kubectl --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1 get prometheus --all-namespaces  
kubectl --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1 get servicemonitor --all-namespaces</pre>

Step 4— Install `subctl` Submariner CLI:

<pre name="c8de" id="c8de" class="graf graf--pre graf-after--p">curl -Ls [https://get.submariner.io](https://get.submariner.io) | bash  
export PATH=$PATH:~/.local/bin  
echo export PATH=\$PATH:~/.local/bin >> ~/.profile</pre>

Step 5 — Deploy Submariner and join clusters to the mesh:  
1) Define Submariner Broker on `cluster1`(I really recommend you to go into the [_Submariner_](https://submariner.io/) website and read about its [architecture](https://submariner.io/getting-started/architecture/) and [the Broker](https://submariner.io/getting-started/architecture/broker/) role in it). In short, one of the Broker roles is to propagate changes in the _Submariner_ datapath to all clusters in the mesh from a central point, avoiding bombarding the system with messages directly between the clusters.  
2) Join `cluster1` and `cluster2` into the mesh

<pre name="0702" id="0702" class="graf graf--pre graf-after--p"># Deploy cluster1 as the Broker cluster  
subctl deploy-broker --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1</pre>

<pre name="529c" id="529c" class="graf graf--pre graf-after--pre"># joins cluster1 and cluster2 into the mesh  
subctl join --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1 broker-info.subm --clusterid cluster1 --natt=false  
subctl join --kubeconfig submariner/output/kubeconfigs/kind-config-  
cluster2 broker-info.subm --clusterid cluster2 --natt=false</pre>

You can now run:

<pre name="2701" id="2701" class="graf graf--pre graf-after--p">kubectl --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1 get servicemonitor --all-namespaces  
kubectl --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1 get servicemonitor --all-namespaces</pre>

And see the relevant `ServiceMonitor` of _Submariner_

<pre name="4290" id="4290" class="graf graf--pre graf-after--p">AMESPACE             NAME                                    AGE  
monitoring            alertmanager                            6h24m  
monitoring            blackbox-exporter                       6h24m  
monitoring            coredns                                 6h24m  
monitoring            grafana                                 6h24m  
monitoring            kube-apiserver                          6h24m  
monitoring            kube-controller-manager                 6h24m  
monitoring            kube-scheduler                          6h24m  
monitoring            kube-state-metrics                      6h24m  
monitoring            kubelet                                 6h24m  
monitoring            node-exporter                           6h24m  
monitoring            prometheus-adapter                      6h24m  
monitoring            prometheus-k8s                          6h24m  
monitoring            prometheus-operator                     6h24m  
submariner-operator   submariner-gateway-metrics              6h22m  
submariner-operator   submariner-lighthouse-agent-metrics     6h22m  
submariner-operator   submariner-lighthouse-coredns-metrics   6h22m  
submariner-operator   submariner-operator-metrics             6h23m</pre>

If for some reason the _Submariner_ `ServiceMonitor`objects are not present you can re-deploy them using

<pre name="e241" id="e241" class="graf graf--pre graf-after--p">kubectl --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1  apply -f submariner-cheatsheet/prometheus/submariner-service-monitors/  
kubectl --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1  apply -f submariner-cheatsheet/prometheus/submariner-service-monitors/</pre>

You can also expose the _Prometheus_ server on one of the clusters and make sure its Targets and Service Discovery is configured correctly

<pre name="1282" id="1282" class="graf graf--pre graf-after--p">kubectl --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1 \  
 port-forward --address 0.0.0.0 svc/prometheus-k8s 9090 --namespace monitoring</pre>

Targets — [http://localhost:9090/targets](http://34.106.143.71:9090/targets)

<figure name="28aa" id="28aa" class="graf graf--figure graf-after--p">![](https://cdn-images-1.medium.com/max/800/1*UOntSNoU731H7ho5w4K15g.png)</figure>

Service Discovery — [http://localhost:9090/service-discovery](http://34.106.143.71:9090/service-discovery)

<figure name="78ee" id="78ee" class="graf graf--figure graf-after--p">![](https://cdn-images-1.medium.com/max/800/1*YDUvY4cvtDFV0Xv0FQLVFg.png)</figure>

I really urge you to go and read about _Prometheus_ and how it utilizes service discovery to discover metrics endpoints. Note that it is important to allow _Prometheus_ access to the `submariner-operator` namespace using ClusterRoleBingins — our `example.jsonnet` file allows for this configuration. If you are using another _Prometheus_ setup like Helm or others you will need to understand how to enable this access role.

Step 6 — Exporting the `Prometheus` server services:

<pre name="7930" id="7930" class="graf graf--pre graf-after--p">subctl export service --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1 --namespace monitoring prometheus-k8s  
subctl export service --kubeconfig submariner/output/kubeconfigs/kind-config-cluster2 --namespace monitoring prometheus-k8s</pre>

This step is using the Kubernetes multi-cluster API implemented by _Submariner_ and exposes the `prometheus-k8s` service in both clusters to be accessed from any cluster in the mesh.

When using _Submariner_ you can access an exported service  
with the following DNS scheme: `service-name.spacename.svc.clusterset.local`.   
Submariner utilizes the Service Discovery mechanized within Kubernetes (by supplying a CoreDNS plugin that monitors changes in the multi-cluster service and supplies the relevant IP address to this DNS query). While the usage of this scheme is the recommended one, right now the implementation of this DNS plugin (named Lighthouse) will prefer to return the IP of the local service (i.e if we query this address from cluster1 and we have that kind of service deployed there it will always return the IP of that service). Alternately, if the service is not deployed on that cluster but some others, currently (September 2021) there is a Round-Robin mechanism that will return a different IP each time.

For these reasons, when we want to access a certain _Prometheus_ server on a certain cluster we will need to use a slightly different scheme: `cluster-name.service-name.namespace.svc.clusterset.local`.  
For example, to access _Prometheus_ metrics endpoint on a cluster named `cluster1`in namespace `monitoring`you can `curl` the following: `curl cluster1.prometheus-k8s.monitoring.svc.clusterset.local:9090\metrics`

Step 7 — Configuring `Grafana` with each of the `Prometheus` servers:

OK, so now we are ready to build some dashboards! There are several different ways to expose the Grafana Service from within the Kubernetes cluster. For our experiment purposes, we will just run in a different shell

<pre name="30ed" id="30ed" class="graf graf--pre graf-after--p">kubectl --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1 port-forward --address 0.0.0.0 svc/grafana 3000 -n monitoring</pre>

Open [http://localohost:3000](http://HTTP://localohost:3000)

1.  Login with admin:admin (user:password)
2.  Add Datasources

<figure name="516e" id="516e" class="graf graf--figure graf-after--li">![](https://cdn-images-1.medium.com/max/800/1*5N1LDjgXi7j8hgDMYTkPwA.png)</figure>

<figure name="490a" id="490a" class="graf graf--figure graf-after--figure">![](https://cdn-images-1.medium.com/max/800/1*affXy5DHVEpNl72B3lmpdg.png)</figure>

Enter the DNS Address with the relevant Prometheus port as explained in the previous section (`cluster1.prometheus-k8s.monitoring.svc.clusterset.local` and `cluster2.prometheus-k8s.monitoring.svc.clusterset.local`)

<figure name="597a" id="597a" class="graf graf--figure graf-after--p">![](https://cdn-images-1.medium.com/max/800/1*XwR_3MUimC1m2Bs1tmvwyA.png)</figure>

Save and test connection (do the following per _Prometheus_ server)

<figure name="76e5" id="76e5" class="graf graf--figure graf-after--p">![](https://cdn-images-1.medium.com/max/800/1*v13k9BaoyDgt7vU7Nwxa3w.png)</figure>

3\. Create your first Dashboard

<figure name="8697" id="8697" class="graf graf--figure graf-after--p">![](https://cdn-images-1.medium.com/max/800/1*TZVCnR09s0FY6y3oCJZ6Cw.png)</figure>

Add an empty panel

<figure name="e632" id="e632" class="graf graf--figure graf-after--p">![](https://cdn-images-1.medium.com/max/800/1*l38QgqAjBxVXc3NfOra2qg.png)</figure>

Define some queries, let’s try and monitor some _Submariner_ metrics. For the full list access [this website](https://submariner.io/operations/monitoring/). Choose the server from the list and add queries, in our example we added simple Submariner metrics for each cluster: `submariner_connections` and `submariner_connection_latency_seconds`

</div>

<div class="section-inner sectionLayout--outsetRow" data-paragraph-count="2">

<figure name="cefb" id="cefb" class="graf graf--figure graf--layoutOutsetRow is-partialWidth graf-after--p" style="width: 49.336%;">![](https://cdn-images-1.medium.com/max/600/1*tjqxxQ_RXMBLx7VEp-dyMA.png)</figure>

<figure name="1ff6" id="1ff6" class="graf graf--figure graf--layoutOutsetRowContinue is-partialWidth graf-after--figure" style="width: 50.664%;">![](https://cdn-images-1.medium.com/max/800/1*HfqFyCKjCLO1oxzcVVxYUw.png)</figure>

</div>

<div class="section-inner sectionLayout--insetColumn">

Step 8 — Optional — Setup `Prometheus Federation:  
`_Prometheus_ offers a federation feature, where one can configure [hierarchical or cross-service federation](https://prometheus.io/docs/prometheus/latest/federation/). The main idea behind it is to allow the aggregation of metrics and information from several different _Prometheus_ servers (possibly across different locations). This is where the _Submariner_ multi-cluster setup comes in handy!

To allow this federation we will need to add additional scraping config to our main _Prometheus server.   
_Take a look at the `submariner-cheatsheet/prometheus/prometheus-additional.yaml`.

<pre name="3570" id="3570" class="graf graf--pre graf-after--p">- job_name: "prometheus-federate"  
  honor_labels: true  
  metrics_path: '/federate'  
  params:  
    match[]: ['{job=~".+"}']  
  static_configs:  
  - targets: ["cluster2.prometheus-k8s.monitoring.svc.clusterset.local"]</pre>

You can see the additional configuration, as described in the federation page of _Prometheus,_ we are adding `cluster2` a static scraping destination. The match labels basically define to scrape all jobs. Take a deeper look into [Prometheus Querying](https://prometheus.io/docs/prometheus/latest/querying/basics/) for more info.

Set `cluster1` as our main monitoring cluster and add the _Prometheus_ federation configuration to it

<pre name="0d3a" id="0d3a" class="graf graf--pre graf-after--p">kubectl --kubeconfig \  
 submariner/output/kubeconfigs/kind-config-cluster1 \  
 create secret generic additional-scrape-configs \  
 --from-file=`submariner-cheatsheet/`prometheus/prometheus-additional.yaml \  
 --dry-run=client -oyaml > \  
 `submariner-cheatsheet/`prometheus/manifests/additional-scrape-configs.yaml</pre>

Note that after running this command a new file will be created under `submariner-cheatsheet/prometheus/manifest/additional-scrape-configs.yaml`.

Now we want to refer to this file from the main _Prometheus_ definition.  
let's edit it:

<pre name="fc72" id="fc72" class="graf graf--pre graf-after--p">nano `submariner-cheatsheet/`prometheus/manifests/prometheus-prometheus.yaml</pre>

Copy and paste the following to the end of the file we just opened:

<pre name="aa79" id="aa79" class="graf graf--pre graf-after--p">additionalScrapeConfigs:  
  name: additional-scrape-configs  
  key: prometheus-additional.yaml</pre>

And apply the new configuration, here we just run the whole manifest but you can run it specifically

<pre name="170c" id="170c" class="graf graf--pre graf-after--p">kubectl --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1 apply -f `submariner-cheatsheet/`prometheus/manifests</pre>

<pre name="bc33" id="bc33" class="graf graf--pre graf-after--pre">or</pre>

<pre name="41d0" id="41d0" class="graf graf--pre graf-after--pre">kubectl --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1 apply -f `submariner-cheatsheet/prometheus/manifest/additional-scrape-configs.yaml`</pre>

<pre name="3893" id="3893" class="graf graf--pre graf-after--pre">kubectl --kubeconfig submariner/output/kubeconfigs/kind-config-cluster1 apply -f `submariner-cheatsheet/`prometheus/manifests/prometheus-prometheus.yaml</pre>

That is it, you can now go to the server targets and see the new federation target configured!

### GitHub Repo

You can go into my [Submariner-Cheatsheet GitHub](https://github.com/danibachar/submariner-cheatsheet) repo, under the `_prometheus_` folder you will see some resources and scripts that can automate the setup.

If for some reason not all of the Submariner `ServiceMonitor` has been deployed, you can re-deploy them using the yaml files supplied under `prometheus/submariner-service-monitor`

</div>

</div>

</section>

</section>

<footer>

By [Daniel Bachar](https://medium.com/@danielbachar) on [<time class="dt-published" datetime="2021-09-28T00:07:53.336Z">September 28, 2021</time>](https://medium.com/p/f89ff733e7ec).

[Canonical link](https://medium.com/@danielbachar/multi-cluster-monitoring-with-prometheus-and-submariner-f89ff733e7ec)

Exported from [Medium](https://medium.com) on September 28, 2021.

</footer>

</article>
