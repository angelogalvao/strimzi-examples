# StreamsHub Console for Apache Kafka with Dex Authentication Layer.

Integrate the StreamsHub Console with Dex Authentication Layer to login in the console using Openshift/Kubernetes credentials.

## Pre requisites

- git client installed
- streams for Apache Kafka installed and available in the cluster
- streams for Apache Kafka Console installed and available in the cluster

NOTE: I am using the supported Red Hat's version os the community projects Strimzi and StramsHub.

## Install Kafka Cluster

Create the Openshift project:
~~~bash
oc new-project kafka-console-test
~~~
Notice that the name of this project will be used bellow. If you choose a different name, be sure to update the commands bellow.

Creaate Kafka the Kafka Cluster and an example topic
~~~bash
oc apply -f 001-kafkanodepool.yaml
oc apply -f 002-kafka-metrics-configmap.yaml
oc apply -f 003-kafka.yaml
oc apply -f 004-kafkatopic.yaml
~~~

## Install Dex

First, clone the latest StreamsHub GutHub repository in your local enviroment:
~~~bash
git clone https://github.com/streamshub/console.git
~~~

Let's use the dex-openshift example by moving to the following folder:
~~~bash
cd examples/dex-openshift
~~~

The README.md file of the folder have detailed instructions, on how to install the Dex. In this guide I will show you the same commands but for the Openshift client and with extra config to run on MacOS. Fell free to use any of the guides:

First let set some variables
~~~bash
export LC_CTYPE=C # This is for MacOS
export LANG=C # This is for MacOS
export NAMESPACE=kafka-console-test
export CLUSTER_DOMAIN=apps..... # MAKE SURE TO CHANGE THAT YOUR CLUSTER DOMAIN
export CLUSTER_APISERVER=$(oc config view --minify=true --flatten=false -o json | jq -r ".clusters[0].cluster.server")
export STATIC_CLIENT_SECRET="$(tr -dc A-Za-z0-9 </dev/urandom | head -c 24)"
~~~

Create a service account for the dex server:
~~~bash
oc create serviceaccount console-dex --dry-run=client -o yaml \
  | oc apply -n ${NAMESPACE} -f -
~~~

Annotate the dex server's service account with the dex redirect URL
~~~bash
oc annotate serviceaccount console-dex -n ${NAMESPACE} \
    --dry-run=client "serviceaccounts.openshift.io/oauth-redirecturi.dex=https://console-dex.${CLUSTER_DOMAIN}/callback" -o yaml \
  | oc apply -n ${NAMESPACE} -f -
~~~

Set secrets used by dex (DEX_CLIENT_ID/SECRET) and the console (DEX_STATIC_CLIENT_SECRET). The token generated below is valid for 1 year, adjust as necessary.
~~~bash
oc create secret generic console-dex-secrets -n ${NAMESPACE} \
  --dry-run=client \
  --from-literal=DEX_CLIENT_ID="system:serviceaccount:${NAMESPACE}:console-dex" \
  --from-literal=DEX_CLIENT_SECRET="$(oc create token -n ${NAMESPACE} console-dex --duration=$((365*24))h)" \
  --from-literal=CONSOLE_SECURITY_OIDC_CLIENT_ID="streamshub-console" \
  --from-literal=CONSOLE_SECURITY_OIDC_CLIENT_SECRET="${STATIC_CLIENT_SECRET}" \
  -o yaml | oc apply -n ${NAMESPACE} -f -
~~~

Edit the file 040-Secret-console-dex.yaml. Look for the **redirectURIs:** line and edit the URL for your StreamsHub console URL. We didn't create the StreamsHub console yet, but the name of the console will be **kafka-console**. So, change the value to accomodate that: 
~~~yaml
# This is just a snippet of the file
      redirectURIs:
      - 'https://kafka-console.${CLUSTER_DOMAIN}/api/auth/callback/oidc'
      - 'http://localhost:3000/api/auth/callback/oidc'
~~~

The install all the resources for Dex by running the following command: 
~~~bash
cat *.yaml | envsubst | oc apply -f - -n ${NAMESPACE}
~~~

## Install StreamsHub Console 

Just install the StreamHub Console. Return to the same folder of this document.
~~~bash
export AUTHSERVER=https://console-dex.$(echo $CLUSTER_DOMAIN)
cat 005-streamshubconsole.yaml | envsubst | oc apply -f - 
~~~

Find out the route URL and test it. 