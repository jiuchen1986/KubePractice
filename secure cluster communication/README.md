# Practice on Securing Communication to Kubernetes Cluster
This part records practices on securing communication to the kubernetes cluster, such as setting up TLS communications to the API server.

## Generate Client Cert&Key to API Server
This section gives approaches generating certs and keys used by customized clients to communicate with Kubernetes API server with specific RBAC authorizations. Steps are given as:

#### Install Cfssl Tools
Download tools from [https://pkg.cfssl.org/](https://pkg.cfssl.org/), i.e. the `cfssl linux-amd64` and the `cfssljson linux-amd`

#### Create RBAC Rules
Create demanding authorizations for specific users or user groups using RBAC in the cluster. For example:


    ### Create a cluster role authorized to read nodes information
    $ kubectl create clusterrole nodes-reader --verb=get,watch,list --resource=nodes -o yaml

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
        creationTimestamp: null
        name: nodes-reader
    rules:
    - apiGroups:
      - ""
      resources:
      - nodes
      verbs:
      - get
      - watch
      - list
    
    ### Bind the created cluster role to a user group
    $ kubectl create clusterrolebinding nodes-reader --clusterrole=nodes-reader --group=monitoring:ganglia -o yaml

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      creationTimestamp: 2018-06-25T06:39:26Z
      name: nodes-reader
      resourceVersion: "9723855"
      selfLink: /apis/rbac.authorization.k8s.io/v1beta1/clusterrolebindings/nodes-reader
      uid: 80e741a1-7842-11e8-8062-f8bc12d8a784
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: nodes-reader
    subjects:
    - apiGroup: rbac.authorization.k8s.io
      kind: Group
      name: monitoring:ganglia

#### Create Key and Initial CSR
Create a initial certificate signing request and the related key for specific users. For example:

    $ cat << EOF | cfssl genkey - | cfssljson -bare nodes-reader
    {
      "CN": "monitoring:ganglia:nodes-reader",
      "key":
      {
        "algo": "rsa",
        "size": 2048
      },
      "names": [
        {
          "O": "monitoring:ganglia"
        }
      ]
    }
    EOF
    
    $ ls
    nodes-reader.csr  nodes-reader-key.pem

Above a CSR and the related key for a user named `nodes-reader` in the group `monitoring:ganglia` are created.

#### Commit CSR to API Server
Create a certificate signing request object using the `.csr` file generated above to send to the Kubernetes API, where the `usages` and the `groups` could be specified. For example:

    $ cat << EOF | kubectl create -f -
    apiVersion: certificates.k8s.io/v1beta1
    kind: CertificateSigningRequest
    metadata:
      name: ganglia-nodes-reader
    spec:
      groups:
      - monitoring:ganglia
      request: $(cat nodes-reader.csr | base64 | tr -d '\n')
      usages:
      - digital signature
      - key encipherment
      - client auth
    EOF

    $ kubectl describe csr ganglia-nodes-reader
    
    Name:               ganglia-nodes-reader
    Labels:             <none>
    Annotations:        <none>
    CreationTimestamp:  Mon, 25 Jun 2018 07:49:25 +0000
    Requesting User:    kubernetes-admin
    Status:             Pending
    Subject:
             Common Name:    monitoring:ganglia:nodes-reader
             Serial Number:  
             Organization:   monitoring:ganglia
    Events:  <none>

#### Approve CSR
Manually approve the csr by admin user.

    $ kubectl certificate approve ganglia-nodes-reader

#### Get Cert
Download the issued certificate from the approved csr and save it to a file.


    $ kubectl get csr ganglia-nodes-reader -o=jsonpath='{.status.certificate}' | base64 -d > nodes-reader.crt

    $ ls
    nodes-reader.crt  nodes-reader.csr  nodes-reader-key.pem

#### Connect to API Server with Cert and Key

