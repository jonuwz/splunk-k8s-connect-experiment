# experiments in splunk-connect-for-kuberetes

### create k8s cluster

we'll use (k3d)[https://k3d.io/] because its simple.  

```bash
k3d cluster create --api-port 6550 -p "8081:80@loadbalancer" test
```

### spin up a splunk container

```bash
kubectl apply -f ./splunk.yaml
```
  
You should be able to get to https://localhost:8081 in a while  

### deploy the splunk-connector-for-kubernetes 

```bash
helm repo add splunk https://splunk.github.io/splunk-connect-for-kubernetes/
helm install splunk-connect -f splunk-k8s-connect.yaml splunk/splunk-connect-for-kubernetes
```
  
There's some interesting configurations in the splunk-k8s-connect.yaml  

``yaml
  customMetadataAnnotations:
    - name: env
      annotation: splunk.com/env
    - name: family
      annotation: splunk.com/family
    - name: app
      annotation: splunk.com/app
    - name: service
      annotation: splunk.com/service

  # customFilters
  customFilters:
    sourceFilter:
      type: jq_transformer
      tag: tail.containers.**
      body: jq '.record.source = .record.splunk_index + ":" + .record.family + ":" + .record.app + ":" + .record.service + ":" + .record.env | .record'
```

This sets the env, family, app and service fields from the annotations, and uses the jq_transformer to add a unique identifier from the concatenation of these fields

### create something that logs

```bash
kubectl apply -f logger_app.yaml
```
  
This creates an app that logs a few events a second in a namespace called `test`
  
By default the events go to the main index, but by setting an aoontation on the namespace, the events will be routed to an index of your choice.
  
```bash
kubectl annotate ns/test splunk.com/index=test
```

### enable terse logging
  
Up to now we've been loggin the full payload - lets clean that up  
```bash
sed -r 's/(^\s+sendAllMetadata:\s*).*/\1false/' splunk-k8s-connect.yaml
helm upgrade splunk-connect -f splunk-k8s-connect.yaml splunk/splunk-connect-for-kubernetes
```

Before 

After
