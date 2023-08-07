## Exposing EAP services though EAP

### Install your service

```shell
helm install eap-demo-service wildfly/wildfly -f helm.yaml
```

### Install 3scale operator

We must use mas operator: http://quay.io/integreatly/3scale-index:v0.11.6-mas (https://github.com/integr8ly/integreatly-operator/blob/master/products/installation.yaml#L63) because the `Application` resource hasn't
been released yet in the Operator version we have on OpenShift:

```shell
cat << EOF | oc create -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: threescalemaslatest
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: quay.io/integreatly/3scale-index:v0.11.6-mas
EOF
```

```shell
export NAMESPACE=3scale-tst-1

cat << EOF | oc create -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  annotations:
  name: $NAMESPACE-operators
  namespace: $NAMESPACE
spec:
  targetNamespaces:
    - $NAMESPACE
  upgradeStrategy: Default
EOF

cat << EOF | oc create -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: 3scale-operator
spec:
  channel: threescale-mas
  installPlanApproval: Automatic
  name: 3scale-operator
  source: threescalemaslatest
  sourceNamespace: openshift-marketplace
  startingCSV: 3scale-operator.v0.11.6-mas
EOF
```

This is taken from [3scale-operator/pull/778](https://github.com/3scale/3scale-operator/pull/778):

```shell
cat << EOF | oc create -f -
apiVersion: apps.3scale.net/v1alpha1
kind: APIManager
metadata:
  name: apimanager-sample
spec:
  system:
    appSpec:
      replicas: 1
    sidekiqSpec:
      replicas: 1
  zync:
    appSpec:
      replicas: 1
    queSpec:
      replicas: 1
  backend:
    cronSpec:
      replicas: 1
    listenerSpec:
      replicas: 1
    workerSpec:
      replicas: 1
  apicast:
    productionSpec:
      replicas: 1
    stagingSpec:
      replicas: 1
  wildcardDomain: $NAMESPACE.apps.eapqe-031-giiq.eapqe.psi.redhat.com
EOF
```

Create a secret to hold credentials of your `DeveloperUser`:

```shell
cat << EOF | oc create -f -
apiVersion: v1
kind: Secret
metadata:
  name: myusername01
stringData:
  password: "123456"
EOF
```

Create a `DeveloperUser` and `DeveloperAccount` to access our API:

```shell
cat << EOF | oc create -f -
apiVersion: capabilities.3scale.net/v1beta1
kind: DeveloperUser
metadata:
  name: developeruser01
spec:
  developerAccountRef:
    name: developeraccount01
  email: myusername01@example.com
  passwordCredentialsRef:
    name: myusername01
  role: admin
  username: myusername01
EOF

cat << EOF | oc create -f -
apiVersion: capabilities.3scale.net/v1beta1
kind: DeveloperAccount
metadata:
  name: developeraccount01
spec:
  orgName: pstefans3
EOF
```

Create a `Backend` for the API provided by EAP:

```shell
cat << EOF | oc create -f -
apiVersion: capabilities.3scale.net/v1beta1
kind: Backend
metadata:
  name: backend1-cr
spec:
  mappingRules:
    - httpMethod: GET
      increment: 1
      last: true
      metricMethodRef: hits
      pattern: /
  name: backend1
  privateBaseURL: 'http://eap-demo-service:8080/api/ping'
  systemName: backend1
EOF
```

Create a `Product` to expose the `Backend`, note that it defines `applicationPlans`: 

```shell
cat << EOF | oc create -f -
apiVersion: capabilities.3scale.net/v1beta1
kind: Product
metadata:
  name: product1-cr
spec:
  applicationPlans:
    plan01:
      name: "My Plan 01"
      limits:
        - period: month
          value: 300
          metricMethodRef:
            systemName: hits
            backend: backend1
    plan02:
      name: "My Plan 02"
      limits:
        - period: month
          value: 300
          metricMethodRef:
            systemName: hits
            backend: backend1
  name: product1
  backendUsages:
    backend1:
      path: /eap-demo-service
EOF
```

Wait for the resources to be `Synced`:

```shell
oc wait --for=condition=Synced --timeout=-1s backend/backend1
oc wait --for=condition=Synced --timeout=-1s product/product1
```

Create an `Application` to link:

```shell
cat << EOF | oc create -f -
apiVersion: capabilities.3scale.net/v1beta1
kind: Application
metadata:
  name: example-application
spec:
  accountCR: 
    name: developeraccount01
  applicationPlanName: plan01
  productCR: 
    name: product1-cr
  name: testApp
  description: testing eap-demo-service with developeraccount01 and plan01
EOF
```

Promote your config:
```shell
cat << EOF | oc create -f -
apiVersion: capabilities.3scale.net/v1beta1
kind: ProxyConfigPromote
metadata:
  name: proxyconfigpromote-sample
spec:
  productCRName: product1-cr
  production: true
  deleteCR: true
EOF
```

Now access Admin Portal with credentials in secret `system-seed`; e.g. https://3scale-admin.$NAMESPACE.apps.eapqe-031-giiq.eapqe.psi.redhat.com/

Go to "Applications" --> "Integration" --> "Configuration", e.g.:

```shell
curl -k "https://product1-3scale-apicast-staging.3scale-tst-1.apps.eapqe-031-giiq.eapqe.psi.redhat.com:443/eap-demo-service?user_key=79a2f2b932b5772bf34b69110361cdf4"
```





















TODO: delete the following one working 

https://access.redhat.com/documentation/en-us/red_hat_3scale_api_management/2.13/html-single/installing_3scale/index

```yaml
apiVersion: apps.3scale.net/v1alpha1
kind: APIManager
metadata:
  name: apimanager-sample
spec:
  system:
    appSpec:
      replicas: 1
    sidekiqSpec:
      replicas: 1
  zync:
    appSpec:
      replicas: 1
    queSpec:
      replicas: 1
  backend:
    cronSpec:
      replicas: 1
    listenerSpec:
      replicas: 1
    workerSpec:
      replicas: 1
  apicast:
    productionSpec:
      replicas: 1
    stagingSpec:
      replicas: 1
  wildcardDomain: 3scale-test.apps.eapqe-031-giiq.eapqe.psi.redhat.com
```

Admin Portal URL: `3scale-admin.3scale-wf.apps.eapqe-031-giiq.eapqe.psi.redhat.com`

- https://access.redhat.com/documentation/en-us/red_hat_3scale_api_management/2.13/html-single/operating_3scale/index#deploying-first-product-backend
- https://github.com/3scale/3scale-operator/pull/778
- https://github.com/3scale/3scale-operator/tree/master/doc/cr_samples

```yaml
apiVersion: capabilities.3scale.net/v1beta1
kind: Backend
metadata:
  name: eap-demo-service-backend
spec:
  name: "eap-demo-service Backend"
  systemName: "eap-demo-service-backend"
  privateBaseURL: "http://eap-demo-service:8080/api/ping"
```

```shell
oc apply -f example/backend.yaml
```

```yaml
apiVersion: capabilities.3scale.net/v1beta1
kind: Product
metadata:
  name: eap-demo-service-product
spec:
  name: "eap-demo-service Product"
  systemName: "eap-demo-service-product"
  backendUsages:
    eap-demo-service-backend:
      path: /eap-demo-service
```

```shell
oc apply -f example/product.yaml

oc wait --for=condition=Synced --timeout=-1s backend/backend1
oc wait --for=condition=Synced --timeout=-1s product/product1
```

```yaml
apiVersion: capabilities.3scale.net/v1beta1
kind: ProxyConfigPromote
metadata:
  name: proxyconfigpromote-sample
spec:
  productCRName: eap-demo-service-product
  production: true
  deleteCR: true
```

```shell
oc apply -f example/proxyConfigPromote.yaml
```

```yaml
apiVersion: capabilities.3scale.net/v1beta1
kind: Application
metadata:
  name: eap-demo-service-application
spec:
  accountCR:
    name: "developeraccount-sample"
  applicationPlanName: "plan01"
  productCR:
    name: "eap-demo-service-product"
  name: "EapDemoApp"
  description: "eap-demo-service testing application "
  suspend: false
```

TODO: 
https://api-3scale-apicast-staging.3scale-wf.apps.eapqe-031-giiq.eapqe.psi.redhat.com/echo-int?user_key=8e6c3b92a8a30d356952b674af193ac9

- email: eastizle+3scaleoperator@redhat.com
  name: Eguzki Astiz Lezaun
- email: mstoklus@redhat.com
  name: Michal Stokluska
- email: aucunnin@redhat.com
  name: Austin Cunningham

### Install upstream 3scale with the Operator

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
