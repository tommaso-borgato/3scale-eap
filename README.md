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
export ADMIN_USER=$(oc get secret/system-seed -o jsonpath='{.data.ADMIN_USER}' | base64 --decode)
echo "ADMIN_USER=$ADMIN_USER"

export ADMIN_PASSWORD=$(oc get secret/system-seed -o jsonpath='{.data.ADMIN_PASSWORD}' | base64 --decode)
echo "ADMIN_PASSWORD=$ADMIN_PASSWORD"

export ADMIN_ACCESS_TOKEN=$(oc get secret/system-seed -o jsonpath='{.data.ADMIN_ACCESS_TOKEN}' | base64 --decode)
echo "ADMIN_ACCESS_TOKEN=$ADMIN_ACCESS_TOKEN"

export ADMIN_URL="https://"$(oc get route -l "zync.3scale.net/route-to=system-provider" -o jsonpath='{.items[0].spec.host}')
echo "ADMIN_URL=$ADMIN_URL"
```

Create a secret to hold credentials of your `DeveloperUser`:

```shell
cat << EOF | oc create -f -
apiVersion: v1
kind: Secret
metadata:
  name: mytenant
type: Opaque
stringData:
  adminURL: $ADMIN_URL
  token: "$ADMIN_ACCESS_TOKEN"
EOF

cat << EOF | oc create -f -
apiVersion: v1
kind: Secret
metadata:
  name: myusername001
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
  name: developeraccount001
spec:
  orgName: eapqe
  providerAccountRef:
    name: mytenant
EOF

cat << EOF | oc create -f -
apiVersion: capabilities.3scale.net/v1beta1
kind: DeveloperUser
metadata:
  name: developeruser001
spec:
  developerAccountRef:
    name: developeraccount001
  email: tborgato@redhat.com
  passwordCredentialsRef:
    name: myusername001
  providerAccountRef:
    name: mytenant
  role: admin
  username: myusername001
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
  # this is optional: it is for having a stable name for the path parameter
  deployment:
    apicastHosted:
      authentication:
        userkey:
          authUserKey: "user_key"
  mappingRules:
    - httpMethod: GET
      increment: 1
      last: true
      metricMethodRef: hits
      pattern: /eap-demo-api
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
  name: eap-demo-application
spec:
  accountCR: 
    name: developeraccount001
  applicationPlanName: plan01
  productCR: 
    name: eap-demo-product-cr
  name: EapDemoApp
  description: testing eap-demo-api with developeraccount001 and plan01
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
curl -k "https://eapdemoproduct-3scale-apicast-staging.3scale3.apps.sultan-7bep.eapqe.psi.redhat.com:443/eap-demo-api?user_key=c13bec2c3c2d7efc697864edfce52134"
```

TODO: get user_key like:

1. Retrieve admin access token from the system-seed secret:
   export THREESCALE_NAMESPACE=<WHERE YOUR 3scale instance lives>
   ADMIN_TOKEN=$(oc get secrets/system-seed -n $THREESCALE_NAMESPACE -o template --template={{.data.ADMIN_ACCESS_TOKEN}} | base64 -d)
   Assumption is that the tenant your are doing it for is the default 3scale tenant. If not the tenant token can be obtained from the secret created during tenant creation (if tenant was created via CR

2. Make an API call to retrieve the applications under that tenant
   curl -X 'GET' "https://<TENANT_NAME>-admin.<DOMAIN>/admin/api/applications.xml?access_token=$ADMIN_TOKEN&page=1&per_page=500" \
   -H 'accept: */*'

For example:
curl -X 'GET' "https://3scale-admin.<YOUR_CLUSTER>/admin/api/applications.xml?access_token=$ADMIN_TOKEN&page=1&per_page=500" \
-H 'accept: */*'

This will return all the applications and the app tokens (the one that's referred to as "user_token" when making a call against apicast). If you only have one app (the default one) you can fetch app token from the output.
If more, find the app ID you are interested in and proceed

3. Fetch the user key of application found by ID
   curl -X 'GET' \
   "https://3scale-admin.<YOUR_CLUSTER>/admin/api/applications/find.xml?access_token=$ADMIN_TOKEN&application_id=5" \
   -H 'accept: */*'
