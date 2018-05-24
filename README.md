# Introduction
This is an extension of the default istio sidecar injector to allow injecting 'perf' container with PID sharing.<br>
The injector can be used in standalone mode for inject 'perf' container in deployed PODs to capture perf statistics for 
performance analysis

# Note
'perf' container requires PID namespace sharing and privilege access. 


## Deploy
- Ensure you are using Kubernetes 1.10+ and the following settings enabled:
  - `PodShareProcessNamespace=true` feature-gate turned on
  - Ensure kube-apiserver has the `admission-control` flag set with `MutatingAdmissionWebhook` and `ValidatingAdmissionWebhook` admission controllers added
```
$kubectl api-versions | grep admissionregistration 

admissionregistration.k8s.io/v1beta1
```

- Download istio-0.7.1 release files
```
$ wget https://github.com/istio/istio/releases/download/0.7.1/istio-0.7.1-linux.tar.gz
$ tar xvf istio-0.7.1-linux.tar.gz
```

- Generate signed cert/key pair 
```
$ ./istio-0.7.1/install/kubernetes/webhook-create-signed-cert.sh \
    --service istio-sidecar-injector \
    --namespace istio-system \
    --secret sidecar-injector-certs
``` 
The resulting cert/key file is stored in the secret `sidecar-injector-certs` for the sidecar injector webhook to consume.

- Install sidecar inject configmap 
```
$ kubectl apply -f perf-sidecar/perf-sidecar-configmap.yaml
```

- Set the `caBundle` in the webhook install YAML that the Kubernetes api-server uses to invoke the webhook
```
$ cat istio-0.7.1/install/kubernetes/istio-sidecar-injector.yaml | \
     ./istio-0.7.1/install/kubernetes/webhook-patch-ca-bundle.sh > \
     istio-0.7.1/install/kubernetes/istio-sidecar-injector-with-ca-bundle.yaml
```

- Install the sidecar injector webhook
```
$ sed -i 's/istio\/sidecar_injector/bpradipt\/sidecar_injector/g' istio-0.7.1/install/kubernetes/istio-sidecar-injector.yaml

$ kubectl apply -f istio-0.7.1/install/kubernetes/istio-sidecar-injector-with-ca-bundle.yaml
```

- Verify if the injector webhook is running
```
$ kubectl -n istio-system get deployment -listio=sidecar-injector

NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
istio-sidecar-injector   1         1         1            1           1d
```

- Label the namespace
The sidecar injector uses NamespaceSelector to decide whether to run the webhook on an object in a namespace. The default webhook configuration uses istio-injection=enabled
The following command sets the label for `default` namespace.
```
$ kubectl label namespace default istio-injection=enabled
```

- Deploy your application
```
$ kubectl apply -f istio-0.7.1/samples/sleep/sleep.yaml 
```

