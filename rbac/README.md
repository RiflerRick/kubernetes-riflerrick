### Authentication and Authorization

In Security in general 2 words come into the picture at the very onset. Authentication and Authorization. 
To check if someone trying to access a system is valid(legit), authentication is required. Once that someone is authenticated, we need to see if that someone is having permissions to carry out certain operations, thats authorization. 

In kubernetes, authentication can be done in many ways, anyone talking to the API server in kubernetes must be authenticated. Every single request made to the API server is authenticated first. During the bootstrapping of the kubernetes cluster itself, the authentication strategy to be followed by the API server must be configured. The most common way of authentication is using X509 client certs for authentication. 

Here is how it works.

Before we dive into how certificates are used in kubernetes for authentication, briefly lets understand what certificates are and how they are used more generally in computers

A public key certificate is basically used to prove the ownership of a public key. The certificate includes:
- information about the key
- the identity of its owner(called the subject)
- the digital signature of an entity that has the verified certificate's contents(issuer)

 If the signature is valid and the software examining the certificate trusts the issuer, then it can use that key to communicate securely with the certificate's subject.  

Here is how certificates are created:
- We first create a private key
  
  ```bash
  openssl genrsa -out rajdeep.key 2048
  ```

- We then create what is called a certificate signing request or CSR

  ```bash
  openssl req -new -key rajdeep.key -out rajdeep.csr -subj "/CN=rajdeep/O=devs"
  ```
  Notice here that we gave a subject where we gave the CN or common name as rajdeep and the O or organization as devs, we will come back to this choice of ours later on as it is significant in the kubernetes world.

- We now need to generate the actual certificate using this CSR
  ```bash
  openssl x509 -req -in rajdeep.csr -CA <the certificate authorities crt file> -CAkey <the certificate authorities key file> -CAcreateserial -out <output filepath of the new crt to be created> -days 500
  ```

  Here note that the signing authority is the CA whose certificate file and private key file I am providing while generating my certificate

  When we use this certificate with the client, the server is going to parse the client certificate to get the certificate authority and if it trusts this certificate authority, it is going to be accepted(aka **authenticated**)

  In case of kubernetes, lets say we are running a minikube cluster in our local machine. In that case, the certificate that is going to be accepted by the api server of minikube is going to be all certificates that are signed by the CA of minikube, in other words that are signed by the crt present in `~/.minikube/ca.crt` and the key present in `~/.minikube/ca.key`. This is an important idea.

Now lets say we want to grant certain users permissions to grant granular access to the kubernetes api server. The way we would do that is to first enable the corresponding user to authenticate with the kube api server. 

Here comes the idea of the common name and the organization while generating the private key. The common name of the private key will basically become the user and the organization becomes the group.

Once we have our new user's certificate, we set these certificates or credentials in the corresponding kubectl context, we use the following command for doing that

```bash
kubectl config set-credentials rajdeep@minikube --client-certificate="$HOME/.certs/kubernetes/minikube/rajdeep.crt" --client-key="$HOME/.certs/kubernetes/minikube/rajdeep.key" --embed-certs=true
```

Here note that the client certificate that was generated was stored in $HOME/.certs/kubernetes/minikube/rajdeep.crt and the client private key was stored in $HOME/.certs/kubernetes/minikube/rajdeep.key

Now we will have set the context to the corresponding cluster
```bash
kubectl config set-context rajdeep@minikube --cluster=minikube --user=rajdeep@minikube
```

Note here that we are using the same user name as the common name with which the certificate private key was generated. this is critical

Once this is done, we are all set, A new context has been created. 

The default permission that is given to a new user is no permissions at all.

This ends the part of authentication, now we can focus on authorization. 

### Authorization

For authorization, there are essentially 3 things in kubernetes:
- subjects, they can be humans or running processes or the like
- API resources, that can be pods, deployments, daemon sets, pvcs, namespaces etc
- verbs: get, list, watch, create, patch, delete etc

In kubernetes, there are roles and role bindings, 

Roles are used to combine API resources to verbs. So for instance, we can create a role called pod-readers that can only perform `list`,  `get` and `watch` operations on the `pod` resource. 

Here is an example of a role:
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: test
  name: pod-access
rules:
  apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```
This above role will allow get and list access of all pods in the test namespace
