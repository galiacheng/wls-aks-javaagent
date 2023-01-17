# Monitor WebLogic on AKS using Azure Application Insight

## Prerequisites

* Azure Subscription. The user should have Owner role in the subscription to deploy the WebLogic on AKS offer.
* Development environment. The sample will run shell commands. You can run the commands in [Azure Cloud Shell](https://learn.microsoft.com/en-us/azure/cloud-shell/overview). If you prefer to run on you local machine, the following tools are required:
    * [Azure CLI](https://docs.microsoft.com/cli/azure); use `az --version` to test if az works. This document was tested with version 2.43.0.
    * [kubectl](https://kubernetes-io-vnext-staging.netlify.com/docs/tasks/tools/install-kubectl/); use `kubectl version` to test if `kubectl` works. This document was tested with version v1.21.1.

## Deploy WebLogic on AKS offer

* Open [WebLogic on AKS offer](https://portal.azure.com/#create/oracle.20210620-wls-on-aks20210620-wls-on-aks) from your browser.
* Fill in the following fileds:
    * **Basics** blade
        * Subscription: select your subscription.
        * Resource group: click Create new, input a name.
        * Region: East US.
        * Username for WebLogic Administrator: `weblogic`.
        * Password for WebLogic Administrator: input a password and confirm the password.
        * Password for WebLogic Model encryption： input a password and confirm the password.
    * **AKS** blade
        * Azure Kubernetes Service
            * 
## Create Azure Application Insight

application insights

InstrumentationKey=662d6802-f3ce-459d-ada2-bab77ab56778;IngestionEndpoint=https://eastus-8.in.applicationinsights.azure.com/;LiveEndpoint=https://eastus.livediagnostics.monitor.azure.com/

## Monitor WebLogic and Applciations

1. Upload java library

-javaagent:"/shared/libs/applicationinsights-agent-3.4.7.jar"

2. 

```bash
WLS_DOMAIN_NS=sample-domain1-ns
WLS_DOMAIN_UID=sample-domain1
APPLICATIONINSIGHTS_CONNECTION_STRING="InstrumentationKey=662d6802-f3ce-459d-ada2-bab77ab56778;IngestionEndpoint=https://eastus-8.in.applicationinsights.azure.com/;LiveEndpoint=https://eastus.livediagnostics.monitor.azure.com/"
AGENT_PATH="-javaagent=/shared/libs/applicationinsights-agent-3.4.7.jar"

JAVA_OPTIONS=$(kubectl -n ${WLS_DOMAIN_NS} get domain ${WLS_DOMAIN_UID} -o json | jq '. | .spec.serverPod.env | .[] | select(.name=="JAVA_OPTIONS") | .value' | tr -d "\"")
JAVA_OPTIONS="\"${AGENT_PATH} ${JAVA_OPTIONS}\""
JAVA_OPTIONS_INDEX=$(kubectl -n ${WLS_DOMAIN_NS} get domain ${WLS_DOMAIN_UID} -o json  | jq '.spec.serverPod.env | map(.name == "JAVA_OPTIONS") | index(true)')
VERSION=$(kubectl -n ${WLS_DOMAIN_NS} get domain ${WLS_DOMAIN_UID} -o json | jq '. | .spec.restartVersion' | tr -d "\"")
VERSION=$((VERSION+1))

cat <<EOF >patch-file.json
[
    {
        "op": "replace",
        "path": "/spec/restartVersion",
        "value": "${VERSION}"
    },
    {
        "op": "remove",
        "path": "/spec/serverPod/env/${JAVA_OPTIONS_INDEX}"
    }
    {
        "op": "add",
        "path": "/spec/serverPod/env/-",
        "value": {
            "name": "APPLICATIONINSIGHTS_CONNECTION_STRING",
            "value": "${APPLICATIONINSIGHTS_CONNECTION_STRING}"
        }
    },
    {
        "op": "add",
        "path": "/spec/serverPod/env/-",
        "value": {
            "name": "JAVA_OPTIONS",
            "value": "${JAVA_OPTIONS}"
        }
    }
]
EOF

kubectl -n ${WLS_DOMAIN_NS} patch domain ${WLS_DOMAIN_UID} \
        --type=json \
        --patch-file patch-file.json

kubectl get pod -n ${WLS_DOMAIN_NS} -w
```

