## Exposing EAP services though 3Scale APIs

### Quick intro to 3scale

Let's say you have a set of internal APIs; for example you might be the owner of a small online business where customers place orders and then you proceed with shipping the orders; in this case, you might need at least the following APIs:
- "Items" API
- "Orders" API
- "Shipments" API 

In `3scale`, "Backends" are used to register each internal API with `3scale` itself:
- "Items" API --> "Items" Backend
- "Orders" API --> "Orders" Backend
- "Shipments" API --> "Shipments" Backend

In `3scale`, a "Product" can be seen as a bundle of APIs with the purpose of serving a specific business goal: ultimately, a Product is what you expose to customers; e.g. placing orders needs access to the "Items" and "Orders" backend APIs:
- "Place Order" Product: includes "Items" Backend + "Orders" Backend

Before publicly exposing a Product, you must define the rules under which the Product can be used: e.g. limit the number of Orders that can be placed in a specific amount of time; In `3scale`, you do this using an "Application Plan"; 
A Product can have one or several Application Plans: for example, a “standard” plan that offers only a set of features with a limit on the number of requests per day/month and a “premium” plan that offers unlimited requests.

In `3scale`, security protocols are defined at the Product API level;

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
echo "NAMESPACE=$NAMESPACE"

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

### Install 3scale and configure Products, Backend etc.

This is taken from [3scale-operator/pull/778](https://github.com/3scale/3scale-operator/pull/778):

```shell
# current namespace
export NAMESPACE=$(oc config view --minify -o 'jsonpath={..namespace}')
echo "NAMESPACE=$NAMESPACE"

# same as web console host but prefixed with namespace
export API_MANAGER_WILDCARD=$(oc get routes/console -n openshift-console -o jsonpath='{.spec.host}' | sed "s/console-openshift-console/$NAMESPACE/")
echo "API_MANAGER_WILDCARD=$API_MANAGER_WILDCARD"

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
  name: eapqetenant
type: Opaque
stringData:
  adminURL: $ADMIN_URL
  token: "$ADMIN_ACCESS_TOKEN"
EOF

cat << EOF | oc create -f -
apiVersion: v1
kind: Secret
metadata:
  name: eapqeusername02
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
  name: eapqedevaccount02
spec:
  orgName: eapqe
  providerAccountRef:
    name: eapqetenant
EOF

cat << EOF | oc create -f -
apiVersion: capabilities.3scale.net/v1beta1
kind: DeveloperUser
metadata:
  name: eapqedevuser02
spec:
  developerAccountRef:
    name: eapqedevaccount02
  email: eapqedevuser02@redhat.com
  passwordCredentialsRef:
    name: eapqeusername02
  providerAccountRef:
    name: eapqetenant
  role: admin
  username: eapqeusername02
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
    standardPlan02:
      name: "Eap-demo-product Standard Plan"
      limits:
        - period: month
          value: 3000
          metricMethodRef:
            systemName: hits
            backend: eap-demo-backend
    premiumPlan02:
      name: "Eap-demo-product Premium Plan"
      limits:
        - period: month
          value: 100000
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
oc wait --for=condition=Synced --timeout=-1s backend/eap-demo-backend-cr
oc wait --for=condition=Synced --timeout=-1s product/eap-demo-product-cr
```

Create an `Application` to link everything together:

```shell
export APPLICATION_NAME=EapDemoApp

cat << EOF | oc create -f -
apiVersion: capabilities.3scale.net/v1beta1
kind: Application
metadata:
  name: eap-demo-application
spec:
  accountCR: 
    name: eapqedevaccount02
  applicationPlanName: standardPlan02
  productCR: 
    name: eap-demo-product-cr
  name: $APPLICATION_NAME
  description: testing eap-demo-api with eapqedevaccount02 and standardPlan02
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

### Invoke your API exposed by 3scale

To invoke your newly exposed API, you need its URL and the `user_key` used to secure it;

You can get them both by accessing the Admin Portal with credentials in secret `system-seed`; e.g. https://3scale-admin.$NAMESPACE.apps.eapqe-031-giiq.eapqe.psi.redhat.com/

Then Go to "Applications" --> "Integration" --> "Configuration" and you'll find everything;

Alternatively, you can do it in an automated way like in the following:

```shell
export ADMIN_ACCESS_TOKEN=$(oc get secret/system-seed -o jsonpath='{.data.ADMIN_ACCESS_TOKEN}' | base64 --decode)
echo "ADMIN_ACCESS_TOKEN=$ADMIN_ACCESS_TOKEN"

export ADMIN_URL="https://"$(oc get route -l "zync.3scale.net/route-to=system-provider" -o jsonpath='{.items[0].spec.host}')
echo "ADMIN_URL=$ADMIN_URL"

curl -k -X 'GET' "$ADMIN_URL/admin/api/applications.xml?access_token=$ADMIN_ACCESS_TOKEN&page=1&per_page=500" -H 'accept: */*' > /tmp/user_key.html
export APPLICATION_NAME=EapDemoApp
export USER_KEY=$(xmllint --xpath "//application[name[text()='$APPLICATION_NAME']]/user_key/text()" /tmp/user_key.html) 

export APICAST_STAGING_URL="https://"$(oc get route -l "zync.3scale.net/route-to=apicast-staging" -o jsonpath='{.items[0].spec.host}')
echo "APICAST_STAGING_URL=$APICAST_STAGING_URL"
   
curl -k "$APICAST_STAGING_URL/eap-demo-api?user_key=$USER_KEY"
```


### Note: general procedure to get the `user_key` automatically

The general procedure to get the `user_key` automatically, is the following:

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
