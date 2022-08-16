# Authorino: K8s-native Zero Trust API Security

Demo of [Authorino](https://github.com/kuadrant/authorino), K8s-native external authorization service, for [DevConf.US 2022](https://devconfus2022.sched.com/event/14Zsq/authorino-k8s-native-zero-trust-api-security), Boston (USA).

You can run this demo in Visual Studio Code as a [Didact Tutorial](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-didact). If you prefer, the demo is also available as a [Jupyter Notebook](./authorino-tutorial.ipynb).

#### Authorino features covered in the demo
- API keys authentication
- OpenID Connect JWT verification
- Authorization based on Kubernetes SubjectAccessReview
- Authorization based on Authorino’s simple JSON pattern-matching authorization policies

## Requirements

- [Docker](https://docker.com)
- [Kind](https://kind.sigs.k8s.io)
- [jq](https://stedolan.github.io/jq)

## The stack

- **Kubernetes cluster**<br/>
  Started locally with [Kind](https://kind.sigs.k8s.io/).
- **News API**<br/>
  Application (REST API) to be protected with Authorino. The following HTTP endpoints are available:
  ```
  POST /{category}          Create a news article
  GET /{category}           List news articles
  GET /{category}/{id}      Read a news article
  DELETE /{category}/{id}   Delete a news article
  ```
- **[Envoy proxy](https://envoyproxy.io)**<br/>
  Deployed as sidecar of the News API, to serve the application with the [External Authorization](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_authz_filter#config-http-filters-ext-authz) filter enabled and pointing to Authorino.
- **[Authorino Operator](https://github.com/kuadrant/authorino-operator)**<br/>
  Cluster-wide installation of the operator and CRDs to manage and use Authorino authorization services.
- **[Authorino](https://github.com/kuadrant/authorino)**<br/>
  The external authorization service, deployed in [`namespaced`](https://github.com/Kuadrant/authorino/blob/main/docs/architecture.md#cluster-wide-vs-namespaced-instances) reconciliation mode, in the same K8s namespace as the News API.
- **[Keycloak server](https://keycloak.org)**<br/>
  (Optional) For a more advanced use case of extending access to the News API for an external group of users managed in the the identity provider.
- **[Contour](https://projectcontour.io)**<br/>
  Kubernetes Ingress Controller based on the Envoy proxy, to handle the ingress traffic to the News API and to Keycloak.

> **Note:** For simplicity, in the demo all components are deployed without TLS.

## Prepare the environment

The following steps are typically performed beforehand by a cluster administrator. They are often out of the scope of the application developer's workflow.

Create the cluster: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kind%20create%20cluster%20--name%20authorino-demo%20--config%20cluster.yaml))

```sh
kind create cluster --name authorino-demo --config cluster.yaml
```

Install Contour: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20apply%20-f%20contour.yaml))

```sh
kubectl apply -f contour.yaml
```

Install the Authorino Operator: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20apply%20-f%20https://raw.githubusercontent.com/Kuadrant/authorino-operator/main/config/deploy/manifests.yaml))

```sh
kubectl apply -f https://raw.githubusercontent.com/Kuadrant/authorino-operator/main/config/deploy/manifests.yaml
```

> **Note:** In OpenShift, the Authorino Operator can alternatively be installed directly from the Red Hat OperatorHub, using [Operator Lifecycle Manager](https://olm.operatorframework.io/).

Install Keycloak: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20apply%20-f%20keycloak.yaml))

```sh
kubectl apply -f keycloak.yaml
```

## Run the demo

### 1. Deploy the News API

The News Agency API ("News API" for short) is minimal. It has no conception of authentication or authorization in its own. Whenever a request hits the API and it is a valid endpoint/operation, it serves the request. If it is a `POST` request to `/{category}`, it creates a news article under that news category. Creating an object here means storing it in memory. There is no persisted database. If it is a GET request to `/{category}`, it  serves the list of news articles in the category, as stored in memory. If it is a `GET` or `DELETE` to `/{category/(article-id}` it serves or deletes the requested object from memory, respectively.

Create the namespace: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20create%20namespace%20news-api))

```sh
kubectl create namespace news-api
```

Deploy the New Agency API in the namespace: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20-n%20news-api%20apply%20-f%20news-api.yaml))

```sh
kubectl -n news-api apply -f news-api.yaml
```

At this point, the News API is running, but it is not protected.

Try the News API unprotected: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20http://news-api.127.0.0.1.nip.io/sports))

```sh
curl http://news-api.127.0.0.1.nip.io/sports
# []
```

### 2. Lock down the News API

Request an instance of Authorino: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20-n%20news-api%20apply%20-f%20authorino.yaml))

```sh
kubectl -n news-api apply -f authorino.yaml
```

> **Note:** With the Authorino Operator running, you can request instances of Authorino deployed cluster-wide (i.e. managing auth definitions across all namespaces in the Kubernetes cluster) or for a particular namespace (i.e., to protect workloads whose auth definitions are defined in the same namespaces as the corresponding Authorino instances themselves). In this demo, we are requesting an Authorino instance in the same namespace as the News API. Cluster-wide Authorino instances are typically setup by cluster administrators beforehand and therefore not part of the developer's workflow.

Add the Envoy sidecar to the News API: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20-n%20news-api%20apply%20-f%20news-api-envoy.yaml))

```sh
kubectl -n news-api apply -f news-api-envoy.yaml
```

<details>
  <summary>Want to see the diff?</summary>

  Check the diff: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$diff%20news-api.yaml%20news-api-envoy.yaml))

  ```sh
  diff news-api.yaml news-api-envoy.yaml
  ```
</details>

<br/>

Try the News API behind Envoy: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20http://news-api.127.0.0.1.nip.io/sports%20-i))

```sh
curl http://news-api.127.0.0.1.nip.io/sports -i
# HTTP/1.1 404 Not Found
# x-ext-auth-reason: Service not found
# server: envoy
# ...
```

### 3. Open up the News API for authenticated and authorized users

Create the AuthConfig: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20-n%20news-api%20apply%20-f%20authconfig-1.yaml))

```sh
kubectl -n news-api apply -f authconfig-1.yaml
```

Try the News API without a valid authentication key: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20http://news-api.127.0.0.1.nip.io/sports%20-i))

```sh
curl http://news-api.127.0.0.1.nip.io/sports -i
# HTTP/1.1 401 Unauthorized
# www-authenticate: APIKEY realm="friends"
# x-ext-auth-reason: credential not found
```

Create an API key: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20-n%20news-api%20apply%20-f%20api-key-1.yaml))

```sh
kubectl -n news-api apply -f api-key-1.yaml
```

Try the News API with a valid API key before granting permission to the user: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20-H%20'Authorization:%20APIKEY%20ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx'%20http://news-api.127.0.0.1.nip.io/sports%20-i))

```sh
curl -H 'Authorization: APIKEY ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx' http://news-api.127.0.0.1.nip.io/sports -i
# HTTP/1.1 403 Forbidden
```

Grant permission to the API key user 'john' in the Kubernetes RBAC: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20-n%20news-api%20apply%20-f%20role.yaml%0Akubectl%20-n%20news-api%20apply%20-f%20rolebinding.yaml))

```sh
kubectl -n news-api apply -f role.yaml
kubectl -n news-api apply -f rolebinding.yaml
```

Try the News API with a valid API key with permission granted to the user: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20-H%20'Authorization:%20APIKEY%20ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx'%20http://news-api.127.0.0.1.nip.io/sports%20-i))

```sh
curl -H 'Authorization: APIKEY ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx' http://news-api.127.0.0.1.nip.io/sports -i
# HTTP/1.1 200 OK
#
# []
```

Try the API for creating a news article: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20-H%20'Authorization:%20APIKEY%20ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx'%20%5C%0A%20%20-X%20POST%20%5C%0A%20%20-d%20'%7B%22title%22:%22Facebook%20shut%20down%20political%20ad%20research,%20daring%20authorities%20to%20pursue%20regulation%22,%22body%22:%22On%20Tuesday,%20Facebook%20stopped%20a%20team%20of%20researchers%20from%20New%20York%20University%20from%20studying%20political%20ads%20and%20COVID-19%20misinformation%20by%20blocking%20their%20personal%20accounts,%20pages,%20apps,%20and%20access%20to%20its%20platform.%20The%20move%20was%20meant%20to%20stop%20NYU%E2%80%99s%20Ad%20Observatory%20from%20using%20a%20browser%20add-on%20it%20launched%20in%202020%20to%20collect%20data%20about%20the%20political%20ads%20users%20see%20on%20Facebook.%20(By%20Christianna%20Silva)%22%7D'%20%5C%0A%20%20http://news-api.127.0.0.1.nip.io/sports))

```sh
curl -H 'Authorization: APIKEY ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx' \
  -X POST \
  -d '{"title":"Facebook shut down political ad research, daring authorities to pursue regulation","body":"On Tuesday, Facebook stopped a team of researchers from New York University from studying political ads and COVID-19 misinformation by blocking their personal accounts, pages, apps, and access to its platform. The move was meant to stop NYU’s Ad Observatory from using a browser add-on it launched in 2020 to collect data about the political ads users see on Facebook. (By Christianna Silva)"}' \
  http://news-api.127.0.0.1.nip.io/sports
```

## Extra: Modify the auth scheme

### 4. Add an external Identity Provider and extra policies

In the [Keycloak Admin Console](http://keycloak.127.0.0.1.nip.io): ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$firefox%20-private-window%20%22http://keycloak.127.0.0.1.nip.io%22))
1. create a realm `devconf`;
2. add users `alice` and `bob` to the realm
    - make sure to mark only Alice's email as verified
    - set a password (`p`) to both users in the _Credentials_ tab
3. create an OpenID Connect client `demo` in the realm.

Modify the AuthConfig: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20-n%20news-api%20apply%20-f%20authconfig-2.yaml))

```sh
kubectl -n news-api apply -f authconfig-2.yaml
```

<details>
  <summary>Want to see the diff?</summary>

  Check the diff: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$diff%20authconfig-1.yaml%20authconfig-2.yaml))

  ```sh
  diff authconfig-1.yaml authconfig-2.yaml
  ```
</details>

<br/>

Grant permission for the Keycloak users 'alice' and 'bob' in the Kubernetes RBAC, by editing the subjects listed in the `RoleBinding`: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$KUBE_EDITOR=%22code%20-w%22%20kubectl%20-n%20news-api%20edit%20rolebinding/news-api))

```sh
KUBE_EDITOR="code -w" kubectl -n news-api edit rolebinding/news-api
```

As Alice, obtain an access token from the Keycloak server: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$ACCESS_TOKEN=$(curl%20http://keycloak.127.0.0.1.nip.io/realms/devconf/protocol/openid-connect/token%20-s%20-d%20'grant_type=password'%20-d%20'client_id=demo'%20-d%20'username=alice'%20-d%20'password=p'%20%7C%20jq%20-r%20.access_token)))

```sh
ACCESS_TOKEN=$(curl http://keycloak.127.0.0.1.nip.io/realms/devconf/protocol/openid-connect/token -s -d 'grant_type=password' -d 'client_id=demo' -d 'username=alice' -d 'password=p' | jq -r .access_token)
```

Try the News API as Alice: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20-H%20%22Authorization:%20Bearer%20$ACCESS_TOKEN%22%20http://news-api.127.0.0.1.nip.io/sports%20-i))

```sh
curl -H "Authorization: Bearer $ACCESS_TOKEN" http://news-api.127.0.0.1.nip.io/sports -i
# HTTP/1.1 200 OK
#
# [{"id":"bff3243f-d9bd-4311-b561-853069e30aca","title":"Facebook shut down political ad research, daring authorities to pursue regulation","body":"On Tuesday, Facebook stopped a team of researchers from New York University from studying political ads and COVID-19 misinformation by blocking their personal accounts, pages, apps, and access to its platform. The move was meant to stop NYU’s Ad Observatory from using a browser add-on it launched in 2020 to collect data about the political ads users see on Facebook. (By Christianna Silva)","date":"2022-08-11 10:55:16 +0000","author":"Unknown","user_id":null}]
```

As Bob, obtain an access token from the Keycloak server: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$ACCESS_TOKEN=$(curl%20http://keycloak.127.0.0.1.nip.io/realms/devconf/protocol/openid-connect/token%20-s%20-d%20'grant_type=password'%20-d%20'client_id=demo'%20-d%20'username=bob'%20-d%20'password=p'%20%7C%20jq%20-r%20.access_token)))

```sh
ACCESS_TOKEN=$(curl http://keycloak.127.0.0.1.nip.io/realms/devconf/protocol/openid-connect/token -s -d 'grant_type=password' -d 'client_id=demo' -d 'username=bob' -d 'password=p' | jq -r .access_token)
```

Try the News API as Bob: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20-H%20%22Authorization:%20Bearer%20$ACCESS_TOKEN%22%20http://news-api.127.0.0.1.nip.io/sports%20-i))

```sh
curl -H "Authorization: Bearer $ACCESS_TOKEN" http://news-api.127.0.0.1.nip.io/sports -i
# HTTP/1.1 403 Forbidden
```

### 5. Inject auth data in the request

Modify the AuthConfig: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20-n%20news-api%20apply%20-f%20authconfig-3.yaml))

```sh
kubectl -n news-api apply -f authconfig-3.yaml
```

<details>
  <summary>Want to see the diff?</summary>

  Check the diff: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$diff%20authconfig-2.yaml%20authconfig-3.yaml))

  ```sh
  diff authconfig-2.yaml authconfig-3.yaml
  ```
</details>

<br/>

Create an article with author: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20-H%20'Authorization:%20APIKEY%20ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx'%20-X%20POST%20%5C%0A%20%20-d%20'%7B%22title%22:%22Biden%20to%20sign%20massive%20climate,%20health%20care%20legislation%22,%22body%22:%22President%20Joe%20Biden%20will%20sign%20Democrats%E2%80%99%20landmark%20climate%20change%20and%20health%20care%20bill%20on%20Tuesday,%20delivering%20what%20he%20has%20called%20the%20%E2%80%9Cfinal%20piece%E2%80%9D%20of%20his%20pared-down%20domestic%20agenda,%20as%20he%20aims%20to%20boost%20his%20party%E2%80%99s%20standing%20with%20voters%20less%20than%20three%20months%20before%20midterm%20elections.%20(By%20The%20Associated%20Press)%22%7D'%20%5C%0A%20%20http://news-api.127.0.0.1.nip.io/politics))

```sh
curl -H 'Authorization: APIKEY ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx' -X POST \
  -d '{"title":"Biden to sign massive climate, health care legislation","body":"President Joe Biden will sign Democrats’ landmark climate change and health care bill on Tuesday, delivering what he has called the “final piece” of his pared-down domestic agenda, as he aims to boost his party’s standing with voters less than three months before midterm elections. (By The Associated Press)"}' \
  http://news-api.127.0.0.1.nip.io/politics
```

List the news articles in the 'politics' category: ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20-H%20'Authorization:%20APIKEY%20ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx'%20http://news-api.127.0.0.1.nip.io/politics))

```sh
curl -H 'Authorization: APIKEY ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx' http://news-api.127.0.0.1.nip.io/politics
```

## Cleanup ([▶︎](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kind%20delete%20cluster%20--name%20authorino-demo))

```sh
kind delete cluster --name authorino-demo
```
