## Exposing EAP services though EAP

### Install your service

```shell
helm install eap-demo-service wildfly/wildfly -f helm.yaml
```

### Install 3scale with the Operator

https://github.com/3scale/3scale-operator/blob/master/doc/operator-user-guide.md

```yaml
apiVersion: apps.3scale.net/v1alpha1
kind: APIManager
metadata:
  name: apimanager-sample
  namespace: 3scale-eap
spec:
  wildcardDomain: 3scale-eap.apps.eapqe-031-giiq.eapqe.psi.redhat.com
```

https://github.com/3scale/3scale-operator/blob/v0.10.1/doc/operator-application-capabilities.md

```yaml
apiVersion: capabilities.3scale.net/v1beta1
kind: Backend
metadata:
  name: eap-demo-service-backend
spec:
  name: "eap-demo-service backend"
  systemName: "eap-demo-service"
  privateBaseURL: "http://eap-demo-service:8080/api/ping"
```

```yaml
apiVersion: capabilities.3scale.net/v1beta1
kind: Product
metadata:
  name: eap-demo-service-product
spec:
  name: "eap-demo-service product"
  systemName: "operated-eap-demo-service-product"
  backendUsages:
    eap-demo-service-backend:
      path: /
```
