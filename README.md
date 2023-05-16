# PROCEDURE DE RESTAURATION D'UN NOEUD OCP4 - SUITE EXPIRATION CERTIFICAT CLIENT DU SVC KUBELET

## Pré-requis

- Client OPENSHIFT "OC" installé 
- Avoir les privilège "cluster-admin"
- Avoir approuvé les Certificats Signing request "CSR" qui sont en attente.
- Etre en possesion de la clé SSH generé dans la phase de deploiement pour se connecté aux differents noeuds



## SCRIPT

Le script permet de  :
-  Vérifier si vous avez les privilèges necessaires,
-  Approuver les demande de csr en cours,
-  Creer le fichier de kubeconfig-bootstrap,

```
#!/bin/bash

set -eou pipefail


####
#### MISE EN FORME
####

LINE_RESET='\e[2K\r'

TEXT_GREEN='\e[032m'
TEXT_YELLOW='\e[33m'
TEXT_RED='\e[31m'
TEXT_RESET='\e[0m'

TEXT_INFO="[${TEXT_YELLOW}i${TEXT_RESET}]"
TEXT_FAIL="[${TEXT_RED}-${TEXT_RESET}]"
TEXT_SUCC="[${TEXT_GREEN}+${TEXT_RESET}]"

echo '   _____ ______ _____  _   _ '
echo '  / ____|  ____|  __ \| \ | |'
echo ' | |    | |__  | |__) |  \| |'
echo ' | |    |  __| |  ___/| . ` |'
echo ' | |____| |____| |    | |\  |'
echo '  \_____|______|_|    |_| \_|'
echo ' PM BOTTERO'


####
#### VERIFICATION DES PRIVILEGES - cluster-admin
####
echo -n -e "${TEXT_INFO} VERIFICATION DES PRIVILEGES (cluster-admin)."
oc whoami | grep 'system:admin' &> /dev/null
if [ $? == 0 ]; then
        echo -n -e "${LINE_RESET}"
        echo -e "${TEXT_SUCC} VERIFICATION DES PRIVILEGES (cluster-admin)."
else
        echo -n -e "${LINE_RESET}"
        echo -e "${TEXT_FAIL} ERREUR, VOUS NAVEZ PAS LES BONS PRIVILEGES"
        exit 255
fi

####
#### APPROUVE ALL CSR
####

echo -n -e "${TEXT_INFO} APPROUVER LES CSR PENDING"
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs --no-run-if-empty oc adm certificate approve
if [ $? -ne 0 ]; then
        echo -n -e "${LINE_RESET}"
        echo -e "${TEXT_FAIL} ERREUR, APPROUVER LES CSR PENDING."
        exit 255
else
        echo -n -e "${LINE_RESET}"
        echo -e "${TEXT_SUCC} APPROUVER LES CSR PENDING."
fi

####
#### DEFINITION DES VARIABLES
####

DIR="$(pwd)"
echo -n -e "${TEXT_INFO} CREATION DES VARIABLES A PARTIR DES ELEMENTS DU CLUSTER."

# RECUPERATION URI DE L'API ET DU CONTEXT
urlapi=$(oc get infrastructures.config.openshift.io cluster -o "jsonpath={.status.apiServerInternalURI}")
context="$(oc config current-context)"

# RECUPERATION INFO CLUSTER EN FCT DU CONTEXT UTILISE
cluster="$(oc config view -o "jsonpath={.contexts[?(@.name==\"$context\")].context.cluster}")"
server="$(oc config view -o "jsonpath={.clusters[?(@.name==\"$cluster\")].cluster.server}")"

# RECUPERATION DES PARAMETTRES D'AUTHENTIFICATION
ca_crt_data="$(oc get secret -n openshift-machine-config-operator node-bootstrapper-token -o "jsonpath={.data.ca\.crt}" | base64 --decode)"
namespace="$(oc get secret -n openshift-machine-config-operator node-bootstrapper-token  -o "jsonpath={.data.namespace}" | base64 --decode)"
token="$(oc get secret -n openshift-machine-config-operator node-bootstrapper-token -o "jsonpath={.data.token}" | base64 --decode)"

if [ $? -ne 0 ]; then
        echo -n -e "${LINE_RESET}"
        echo -e "${TEXT_FAIL} ERREUR, RECUPERATION DES VARIABLES DU CLUSTER."
        exit 255
else
        echo -n -e "${LINE_RESET}"
        echo -e "${TEXT_SUCC} RECUPERATION DES VARIABLES DU CLUSTER."
fi

####
#### CREATION DU FICHIER KUBECONFIG TEMPORAIRE VIA (MKTEMP)
####

echo -n -e "${TEXT_INFO} CREATION D'UN FICHIER KUBECONFIG TEMPORAIRE."
export KUBECONFIG="$(mktemp)"
if [ $? -ne 0 ]; then
        echo -n -e "${LINE_RESET}"
        echo -e "${TEXT_FAIL} ERREUR, CREATION D'UN FICHIER KUBECONFIG TEMPORAIRE."
        exit 255
else
        echo -n -e "${LINE_RESET}"
        echo -e "${TEXT_SUCC} CREATION D'UN FICHIER KUBECONFIG TEMPORAIRE. ( ${KUBECONFIG})"
fi

####
#### DEFINITION DU TOKEN BOOTSTRAP POUR LE COMPTE KUBELET
####

echo -n -e "${TEXT_INFO} SET TOKEN BOOTSTRAP POUR LE COMPTE KUBELET."
oc config set-credentials "kubelet" --token="$token" >/dev/null
if [ $? -ne 0 ]; then
        echo -n -e "${LINE_RESET}"
        echo -e "${TEXT_FAIL} ERREUR, SET TOKEN BOOTSTRAP POUR LE COMPTE KUBELET."
        rm -rf ${KUBECONFIG}
        exit 255
else
        echo -n -e "${LINE_RESET}"
        echo -e "${TEXT_SUCC} SET TOKEN BOOTSTRAP POUR LE COMPTE KUBELET."
fi

####
#### COPIE DU CERTIFICAT DE LA CA DU CLUSTER
####

echo -n -e "${TEXT_INFO} COPIE CERT CA ROOT."
ca_crt="$(mktemp)"; echo "$ca_crt_data" > $ca_crt
if [ $? -ne 0 ]; then
        echo -n -e "${LINE_RESET}"
        echo -e "${TEXT_FAIL} ERREUR, COPIE CERT CA ROOT."
        rm -rf ${KUBECONFIG} ${ca_crt}
        exit 255
else
        echo -n -e "${LINE_RESET}"
        echo -e "${TEXT_SUCC} COPIE CERT CA ROOT.( $ca_crt )"
fi

####
#### CONFIGURATION DU CONTEXT ASSOCIE AU COMPTE KUBELET
####

echo -n -e "${TEXT_INFO} CONFIGURATION DU CONTEXT ASSOCIE AU COMPTE KUBELET."
oc config set-cluster $cluster --server="$urlapi" --certificate-authority="$ca_crt" --embed-certs >/dev/null
oc config set-context kubelet --cluster="$cluster" --user="kubelet" >/dev/null
oc config use-context kubelet >/dev/null
if [ $? -ne 0 ]; then
        echo -n -e "${LINE_RESET}"
        echo -e "${TEXT_FAIL} ERREUR, CONFIGURATION DU CONTEXT ASSOCIE AU COMPTE KUBELET."
        rm -rf ${KUBECONFIG} ${ca_crt}
        exit 255
else
        echo -n -e "${LINE_RESET}"
        echo -e "${TEXT_SUCC} CONFIGURATION DU CONTEXT ASSOCIE AU COMPTE KUBELET."
fi

####
#### COPIE VERS LE REPETOIRE COURANT
####

echo -n -e "${TEXT_INFO} COPIE VERS LE REPETOIRE COURANT."
mv "$KUBECONFIG" ${DIR}/kubeconfig-bootstrap
if [ $? -ne 0 ]; then
        echo -n -e "${LINE_RESET}"
        echo -e "${TEXT_FAIL} ERREUR, COPIE VERS LE REPETOIRE COURANT."
        rm -rf ${KUBECONFIG} ${ca_crt}
        exit 255
else
        echo -n -e "${LINE_RESET}"
        echo -e "${TEXT_SUCC} COPIE VERS LE REPETOIRE COURANT. ( ${DIR}/kubeconfig-bootstrap})"
fi

####
#### CHANGEMENT DE DROITS
####

echo -n -e "${TEXT_INFO} CHANGEMENT DE DROIT - RWX RX RX"
chmod 755 ${DIR}/kubeconfig-bootstrap
if [ $? -ne 0 ]; then
        echo -n -e "${LINE_RESET}"
        echo -e "${TEXT_FAIL} ERREUR, CHANGEMENT DE DROIT - RWX RX RX."
        rm -rf ${KUBECONFIG} ${ca_crt}
        exit 255
else
        echo -n -e "${LINE_RESET}"
        echo -e "${TEXT_SUCC} CHANGEMENT DE DROIT - RWX RX RX. ( ${DIR}/kubeconfig-bootstrap})"
fi

echo -e "${TEXT_SUCC} FIN DU SCRIPT. SUPPRESION DES FICHIERS TEMPORAIRES."
rm -rf ${KUBECONFIG} ${ca_crt}

```


## PROCEDURE A REALISER SUR LE NOEUD INCRIMINE

- Arret du services "kubelet"
- Sauvegarde des certificats et fichier kubeconfig
- Suppresion des elements existants
- Transfert du fichier kubeconfig-bootstrap

### Prise a distance du noeud

```
ssh -i ~/.ssh/key_ssh core@addresse_ip_noeud
```
###
```
# systemctl stop kubelet
# mkdir -p /root/sauvegarde-certs
# cp -a /var/lib/kubelet/pki /var/lib/kubelet/kubeconfig /root/sauvegarde-certs
# rm -rf /var/lib/kubelet/pki /var/lib/kubelet/kubeconfig`
```
### Transfert du fichier de restauration

```
scp -i ~/.ssh/key_ssh ./kubeconfig-bootstrap core@addresse_ip_noeud:/etc/kubernetes/kubeconfig
```

### redemarrage  du service 

```
# systemctl start kubelet
```

## VALIDATION DE LA NOUVELLE CSR

- Validation de la nouvelle CSR

```
oc get-csr
oc adm certificate approve csr-xyz
```

Après quelques minutes le noeud devrait être operationnel "READY"