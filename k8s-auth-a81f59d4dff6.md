Unknown markup type 10 { type: [33m10[39m, start: [33m65[39m, end: [33m86[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m52[39m, end: [33m71[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m50[39m, end: [33m65[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m122[39m, end: [33m131[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m46[39m, end: [33m110[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m14[39m, end: [33m41[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m28[39m, end: [33m64[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m9[39m, end: [33m18[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m23[39m, end: [33m36[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m40[39m, end: [33m61[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m66[39m, end: [33m85[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m19[39m, end: [33m35[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m22[39m, end: [33m51[39m }

# Kubernetes authentication via GitHub OAuth and Dex

Here’s a step-by-step guide for generating kubectl credentials using Dex, dex-k8s-authenticator and GitHub.

![Unfortunately, it’s not in Kubernetes vanilla](https://cdn-images-1.medium.com/max/2000/1*OfO_kV-hMsUfokKTOfRHKQ.png)*Unfortunately, it’s not in Kubernetes vanilla*

My name is Amet Umerov and I’m a DevOps Engineer at [Preply](https://preply.com/en/).

## Introduction to Kubernetes auth

We use Kubernetes for creating dynamic environments for devs and QA. So we want to provide them access to Kubernetes via Dashboard and CLI. Kubernetes vanilla doesn’t support authentication for kubectl out of the box, unlike OpenShift.

In this configuration example, we use:

* [dex-k8s-authenticator](https://github.com/mintel/dex-k8s-authenticator) — a web application for generating kubectl config

* [Dex](https://github.com/dexidp/dex) — OpenID Connect provider

* GitHub — because we use GitHub at our company

Unfortunately, [Dex can’t handle](https://github.com/dexidp/dex/issues/1065) groups with Google OIDC, so if you want to use groups, try another provider. Without groups, you can’t create group-based RBAC policies.

Here is a flow of how Kubernetes authorization works:

![The Authorization process](https://cdn-images-1.medium.com/max/7150/1*cGOhqcGOsKw6yG6Oljucyw.png)*The Authorization process*

1. The user initiates a login request in the dex-k8s-authenticator (login.k8s.example.com)

1. dex-k8s-authenticator redirects the request to Dex (dex.k8s.example.com)

1. Dex redirects to the GitHub authorization page

1. GitHub encrypts the corresponding information and passes it back to Dex

1. Dex forwards this information to dex-k8s-authenticator

1. The user gets the OIDC token from GitHub

1. dex-k8s-authenticator adds the token to kubeconfig

1. kubectl passes the token to KubeAPIServer

1. KubeAPIServer returns the result to kubectl

1. The user gets the information from kubectl

## Prerequisites

So, we have already installed Kubernetes cluster (k8s.example.com) and HELM. Also, we have GitHub with organization name (super-org).

If you don’t have HELM, you can easily [install it](https://docs.helm.sh/using_helm/#installing-helm).

Go to the GitHub organization Settings page, (https://github.com/organizations/super-org/settings/applications) and create a new Authorized OAuth App:

![The GitHub settings page](https://cdn-images-1.medium.com/max/2152/1*4KAGf71GTJzEt_RExdPo4Q.png)*The GitHub settings page*

Fill the fields with your values:

* Homepage URL: https://dex.k8s.example.com

* Authorization callback URL: https://dex.k8s.example.com/callback

*Be careful with links, trailing slashes are important.*

Save the Client ID and Client secret generated by GitHub in a safe place (we use [Vault](https://www.vaultproject.io/) for storing our secrets):

    Client ID: **1ab2c3d4e5f6g7h8**
    Client secret: **98z76y54x32w1**

Prepare your DNS records for subdomains login.k8s.example.com and dex.k8s.example.com and SSL certificates for Ingress.

Create SSL certificates:

    cat <<EOF | kubectl create -f -
    apiVersion: certmanager.k8s.io/v1alpha1
    kind: Certificate
    metadata:
      name: **cert-auth-dex**
      namespace: kube-system
    spec:
      secretName: **cert-auth-dex**
      dnsNames:
        - **dex.k8s.example.com**
      acme:
        config:
        - http01:
            ingressClass: nginx
          domains:
          - **dex.k8s.example.com**
      issuerRef:
        name: **le-clusterissuer**
        kind: ClusterIssuer
    ---
    apiVersion: certmanager.k8s.io/v1alpha1
    kind: Certificate
    metadata:
      name: **cert-auth-login**
      namespace: kube-system
    spec:
      secretName: **cert-auth-login**
      dnsNames:
        - **login.k8s.example.com**
      acme:
        config:
        - http01:
            ingressClass: nginx
          domains:
          - **login.k8s.example.com**
      issuerRef:
        name: **le-clusterissuer**
        kind: ClusterIssuer
    EOF

    kubectl describe certificates cert-auth-dex -n kube-system
    kubectl describe certificates cert-auth-login -n kube-system

Your ClusterIssuer le-clusterissuer should already exist, if you haven’t done it you can easily create it via HELM:

    helm install --namespace kube-system -n cert-manager stable/cert-manager

    cat << EOF | kubectl create -f -
    apiVersion: certmanager.k8s.io/v1alpha1
    kind: ClusterIssuer
    metadata:
      name: **le-clusterissuer**
      namespace: kube-system
    spec:
      acme:
        server: [https://acme-v02.api.letsencrypt.org/directory](https://acme-v02.api.letsencrypt.org/directory)
        email: **k8s-admin@example.com**
        privateKeySecretRef:
          name: **le-clusterissuer**
        http01: {}
    EOF

## **KubeAPIServer setup**

You need to provide OIDC configuration for the kubeAPIServer as below and update cluster:

    kops edit cluster
    ...
      kubeAPIServer:
        anonymousAuth: false
        authorizationMode: RBAC
        oidcClientID: dex-k8s-authenticator
        oidcGroupsClaim: groups
        oidcIssuerURL: **https://dex.k8s.example.com/**
        oidcUsernameClaim: email

    kops update cluster --yes
    kops rolling-update cluster --yes

In our case, we use [kops](https://github.com/kubernetes/kops) for cluster provisioning but it works the same for [other clusters](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/).

## **Dex and dex-k8s-authenticator setup**

For connecting Dex you should have a Kubernetes certificate and key. Let’s obtain from the master:

    sudo cat /srv/kubernetes/ca.{crt,key}
    -----BEGIN CERTIFICATE-----
    AAAAAAAAAAABBBBBBBBBBCCCCCC
    -----END CERTIFICATE-----

    -----BEGIN RSA PRIVATE KEY-----
    DDDDDDDDDDDEEEEEEEEEEFFFFFF
    -----END RSA PRIVATE KEY-----

Clone the dex-k8s-authenticator repo:

    git clone git@github.com:mintel/dex-k8s-authenticator.git
    cd dex-k8s-authenticator/

You can set up a values file very flexible, [HELM charts](https://github.com/mintel/dex-k8s-authenticator/tree/master/charts) are available on GitHub. Dex will not work with default variables.

Create the values file for Dex:

    cat << \EOF > values-dex.yml
    global:
      deployEnv: prod

    tls:
      certificate: |-
    **    -----BEGIN CERTIFICATE-----
        AAAAAAAAAAABBBBBBBBBBCCCCCC
        -----END CERTIFICATE-----**
      key: |-
    **    -----BEGIN RSA PRIVATE KEY-----
        DDDDDDDDDDDEEEEEEEEEEFFFFFF
        -----END RSA PRIVATE KEY-----**

    ingress:
      enabled: true
      annotations:
        kubernetes.io/ingress.class: nginx
        kubernetes.io/tls-acme: "true"
      path: /
      hosts:
        - **dex.k8s.example.com**
      tls:
        - secretName: **cert-auth-dex**
          hosts:
            - **dex.k8s.example.com**

    serviceAccount:
      create: true
      name: dex-auth-sa

    config: |
      issuer: **https://dex.k8s.example.com/**
      storage: *# [https://github.com/dexidp/dex/issues/798](https://github.com/dexidp/dex/issues/798)*
        type: sqlite3
        config:
          file: /var/dex.db
      web:
        http: 0.0.0.0:5556
      frontend:
        theme: "coreos"
        issuer: "Example Co"
        issuerUrl: "https://example.com"
        logoUrl: https://example.com/images/logo-250x25.png
      expiry:
        signingKeys: "6h"
        idTokens: "24h"
      logger:
        level: debug
        format: json
      oauth2:
        responseTypes: ["code", "token", "id_token"]
        skipApprovalScreen: true
      connectors:
      - type: github
        id: github
        name: GitHub
        config:
          clientID: $GITHUB_CLIENT_ID
          clientSecret: $GITHUB_CLIENT_SECRET
          redirectURI: **https://dex.k8s.example.com/callback**
          orgs:
          - name: **super-org
    **        teams:
            - **team-red**

    **  **staticClients:
      - id: dex-k8s-authenticator
        name: dex-k8s-authenticator
        secret: **generatedLongRandomPhrase
        **redirectURIs:
          - **https://login.k8s.example.com/callback/**

    envSecrets:
      GITHUB_CLIENT_ID: **"1ab2c3d4e5f6g7h8"**
      GITHUB_CLIENT_SECRET: **"98z76y54x32w1"
    **EOF

And for dex-k8s-authenticator:

    cat << EOF > values-auth.yml
    global:
      deployEnv: prod

    dexK8sAuthenticator:
      clusters:
      - name: k8s.example.com
        short_description: "k8s cluster"
        description: "Kubernetes cluster"
        issuer: **https://dex.k8s.example.com/**
        k8s_master_uri: **https://api.k8s.example.com**
        client_id: dex-k8s-authenticator
        client_secret: **generatedLongRandomPhrase**
        redirect_uri: **https://login.k8s.example.com/callback/**
        k8s_ca_pem: |
    **      -----BEGIN CERTIFICATE-----
          AAAAAAAAAAABBBBBBBBBBCCCCCC
          -----END CERTIFICATE-----**

    ingress:
      enabled: true
      annotations:
        kubernetes.io/ingress.class: nginx
        kubernetes.io/tls-acme: "true"
      path: /
      hosts:
        - **login.k8s.example.com**
      tls:
        - secretName: **cert-auth-login**
          hosts:
            - **login.k8s.example.com
    **EOF

Install Dex and dex-k8s-authenticator:

    helm install -n dex --namespace kube-system --values values-dex.yml charts/dex
    helm install -n dex-auth --namespace kube-system --values values-auth.yml charts/dex-k8s-authenticator

Check it (Dex should return code 400, and dex-k8s-authenticator — code 200):

    curl -sI https://dex.k8s.example.com/callback | head -1
    HTTP/2 400

    curl -sI https://login.k8s.example.com/ | head -1
    HTTP/2 200

## RBAC configuration

Create ClusterRole for your group, in our case with read-only permissions:

    cat << EOF | kubectl create -f -
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: **cluster-read-all**
    rules:
      -
        apiGroups:
          - ""
          - apps
          - autoscaling
          - batch
          - extensions
          - policy
          - rbac.authorization.k8s.io
          - storage.k8s.io
        resources:
          - componentstatuses
          - configmaps
          - cronjobs
          - daemonsets
          - deployments
          - events
          - endpoints
          - horizontalpodautoscalers
          - ingress
          - ingresses
          - jobs
          - limitranges
          - namespaces
          - nodes
          - pods
          - pods/log
          - pods/exec
          - persistentvolumes
          - persistentvolumeclaims
          - resourcequotas
          - replicasets
          - replicationcontrollers
          - serviceaccounts
          - services
          - statefulsets
          - storageclasses
          - clusterroles
          - roles
        verbs:
          - get
          - watch
          - list
      - nonResourceURLs: ["*"]
        verbs:
          - get
          - watch
          - list
      - apiGroups: [""]
        resources: ["pods/exec"]
        verbs: ["create"]
    EOF

Create ClusterRoleBinding configuration:

    cat <<EOF | kubectl create -f -
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRoleBinding
    metadata:
      name: **dex-cluster-auth**
      namespace: kube-system
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: **cluster-read-all**
    subjects:
      kind: Group
      name: **"super-org:team-red"
    **EOF

Now you are ready to start testing.

## Tests

Go to the login page (https://login.k8s.example.com) and sign in with your GitHub account:

![Login page](https://cdn-images-1.medium.com/max/2728/1*EZQM2p3ubSv4YogXiJ9ujQ.png)*Login page*

![Login page redirected to GitHub](https://cdn-images-1.medium.com/max/4468/1*DaxoB2Y1U4Qzj_qSK5V7Ow.png)*Login page redirected to GitHub*

![Follow the instructions to create kubectl config](https://cdn-images-1.medium.com/max/4468/1*QKHZl_v-s2T_HJ5VYdXdrg.png)*Follow the instructions to create kubectl config*

After copy-pasting commands from the login web page you can use kubectl with your cluster:

    kubectl get po
    NAME                READY   STATUS    RESTARTS   AGE
    mypod               1/1     Running   0          3d

    kubectl delete po mypod
    Error from server (Forbidden): pods "mypod" is forbidden: User "amet@example.com" cannot delete pods in the namespace "default"

And it works! All users with GitHub account in our organization can read resources and exec into pods but have no write permissions.

*Stay tuned and subscribe to our blog, we will publish new articles soon :)*
