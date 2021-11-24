# Kubernetes Cert Manager Webhook for Name.com

Cert Manager Webhook for Name.com is an [ACME webhook solver](https://cert-manager.io/docs/configuration/acme/dns01/webhook/) for [cert-manager](https://cert-manager.io/) that enables the use of DNS01 challenges with [Name.com](https://name.com/) as the DNS provider, via the [Name.com API](https://www.name.com/api-docs).

## :toolbox: Requirements

- [go](https://golang.org/) `>= 1.13.0`
- [helm](https://helm.sh/) `>= v3.0.0` [installed](https://helm.sh/docs/intro/install/) on your computer
- [kubernetes](https://kubernetes.io/) `>= v1.14.0` (`>=v1.19` recommended)
- [cert-manager](https://cert-manager.io/) `>= 0.12.0` [installed](https://cert-manager.io/docs/installation/) on the cluster
- A Name.com account with a [Name.com v4 Production API token](https://www.name.com/support/articles/360007597874-signing-up-for-api-access)
- A valid domain registered and [configured with Name.com's default nameservers](https://www.name.com/support/articles/205188648-using-name-coms-default-nameservers)

## :package: Installation

### 1. Webhook

Use a local checkout of this repository and install the webhook with Helm:

```shell{:copy}
helm install --namespace cert-manager cert-manager-webhook-namecom  ./deploy/cert-manager-webhook-namecom/
```

> :bell: **Note:** The webhook should be deployed into the same namespace as `cert-manager`. If you changed that, you should update the `certManager.namespace` value in the deploy template file, [`values.yaml`](deploy/cert-manager-webhook-namecom/values.yaml), before installation.

#### Uninstallation

You can also remove the webhook using Helm:

```shell{:copy}
helm uninstall --namespace cert-manager cert-manager-webhook-namecom
```

### 2. API token secret

Create a [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/) to store the value of your Name.com API token:

```shell{:copy}
kubectl create secret generic namedotcom-credentials --from-literal=api-token=<your API token> --namespace cert-manager 
```

> :bulb: **Note:** The secret should also be in the same namespace as `cert-manager`. If you change the name of the secret or key, don't forget to use those values in the Issuer below.

### 3. Certificate issuer

Define a [cert-manager Issuer (or ClusterIssuer)](https://cert-manager.io/docs/concepts/issuer/) resource that uses the webhook as the solver. Create a file called, e.g. `cert-issuer.yml`, and use the following content as the template:

###### `cert-issuer.yml`
```yaml{:copy}
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-namecom
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # For production use this URL instead:
    # server: https://acme-v02.api.letsencrypt.org/directory
    email: <you@your-email-address.com>
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
    - dns01:
        webhook:
          groupName: acme.name.com
          solverName: namedotcom
          config:
            username: <your Name.com username>
            apitokensecret:
              name: namedotcom-credentials
              key: api-token
```

> :bulb: **Note:** The `config` key for the webhook defines your Name.com API credentials—the `apitokensecret.name` and `apitokensecret.key` values must match those for your secret, above.

Apply the file to your cluster to create the resource:

```shell{:copy}
kubectl apply -f cert-issuer.yaml
```

> :bell:**Note:** If you defined an `Issuer` rather than a `ClusterIssuer`, you should create it in the same namespace as `cert-manager`.

## :scroll: Issue a certificate

Create a certificate by defining a [cert-manager Certificate](https://cert-manager.io/docs/concepts/certificate/) resource and applying it to your cluster:

###### `example-cert.yml`
```yaml{:copy}
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
spec:
  dnsNames:
    - test.example.com
  issuerRef:
    name: letsencrypt-namecom
    kind: ClusterIssuer
  secretName: example-cert
```

> :bulb: **Note:** If you defined an `Issuer` rather than a `ClusterIssuer`, you can omit the `issuerRef.kind` key.

```shell{:copy}
kubectl apply -f example-cert.yml
```

After allowing a short period for the challenge, order and issuing process to complete, the certificate should be available for use: :partying_face:

```shell
$ kubectl get certificate example-com
NAME          READY   SECRET             AGE
example-com   True    example-com-cert   1m12s
```

## :wrench: Development

### Running the test suite

All DNS providers **must** run the DNS01 provider conformance testing suite,
else they will have undetermined behaviour when used with cert-manager.

> :heavy_check_mark: **It is essential that you configure and run the test suite when creating a
DNS01 webhook.**

Before running the test suite, you must supply valid credentials for the Name.com API. See the [test data README](testdata/namedotcom/README.md) for more information.

You can run the test suite with:

```bash
TEST_ZONE_NAME=example.com. make test
```

> :bell: **Note:** `example.com` must also be a domain registered to your Name.com and [configured with Name.com's default nameservers](https://www.name.com/support/articles/205188648-using-name-coms-default-nameservers) so that DNS records can be [managed via Name.com DNS](https://www.name.com/support/categories/200296808-domains).
