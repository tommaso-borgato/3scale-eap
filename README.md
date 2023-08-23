## Exposing EAP services though EAP

### Install your service

```shell
helm install eap-demo-service wildfly/wildfly -f helm.yaml
```

After installation completes, your application is available at e.g.:

```shell
$ curl -k https://eap-demo-service-3scale1.apps.sultan-7bep.eapqe.psi.redhat.com/api/ping
<html><body><h3>Ping Service Servlet</h3></body></html>
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
# current namespace
export NAMESPACE=$(oc config view --minify -o 'jsonpath={..namespace}')

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
# current namespace
export NAMESPACE=$(oc config view --minify -o 'jsonpath={..namespace}')
# same as web console host but prefixed with namespace
export API_MANAGER_WILDCARD=$(oc get routes/console -n openshift-console -o jsonpath='{.spec.host}' | sed "s/console-openshift-console/$NAMESPACE/")

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
  wildcardDomain: $API_MANAGER_WILDCARD
EOF
```

You can retrieve the credentials to access your 3Scale admin GUI (e.g. https://3scale-admin.3scale1.apps.eapqe-031-giiq.eapqe.psi.redhat.com) with the following:

```shell
oc get secret/system-seed -o jsonpath='{.data.ADMIN_USER}' | base64 --decode
oc get secret/system-seed -o jsonpath='{.data.ADMIN_PASSWORD}' | base64 --decode
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
kind: DeveloperAccount
metadata:
  name: developeraccount01
spec:
  orgName: eapqe
EOF

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
```

Create a `Backend` for the API provided by EAP:

```shell
cat << EOF | oc create -f -
apiVersion: capabilities.3scale.net/v1beta1
kind: Backend
metadata:
  name: eap-demo-backend-cr
spec:
  name: "Eap-demo Backend"
  privateBaseURL: 'http://eap-demo-service:8080/api/ping'
  systemName: eap-demo-backend
EOF
```

Create a `Product` to expose the `Backend`, note that `Product` contains `applicationPlans`: 

```shell
cat << EOF | oc create -f -
apiVersion: capabilities.3scale.net/v1beta1
kind: Product
metadata:
  name: eap-demo-product-cr
spec:
  applicationPlans:
    plan01:
      name: "Eap-demo-product Plan"
      limits:
        - period: month
          value: 3000
          metricMethodRef:
            systemName: hits
            backend: eap-demo-backend
  name: eap-demo-product
  backendUsages:
    eap-demo-backend:
      path: /eap-demo-api
EOF
```

Wait for the resources to be `Synced`:

```shell
oc wait --for=condition=Synced --timeout=-1s backend/eap-demo-backend
oc wait --for=condition=Synced --timeout=-1s product/eap-demo-product
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
    name: eap-demo-product-cr
  name: testApp
  description: testing eap-demo-api with developeraccount01 and plan01
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
  productCRName: eap-demo-product-cr
  production: true
  deleteCR: true
EOF
```

Now access Admin Portal with credentials in secret `system-seed`; e.g. https://3scale-admin.$NAMESPACE.apps.eapqe-031-giiq.eapqe.psi.redhat.com/

Go to "Applications" --> "Integration" --> "Configuration", e.g.:

```shell
curl -k "https://eap-demo-product-3scale-apicast-staging.3scale-tst-1.apps.eapqe-031-giiq.eapqe.psi.redhat.com:443/eap-demo-api?user_key=79a2f2b932b5772bf34b69110361cdf4"
```
