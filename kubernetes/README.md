# Quickstart for Kubernetes
This quickstart provides step by step instructions for deploying a Grey Matter service mesh on Kubernetes.

## Contents

- [Overview](#overview)
- [Dependencies](#dependencies)
- [Instructions](#instructions)
  - [Setup](#setup)
  - [Deploying SPIRE](#spire)
  - [Deploying Grey Matter Fabric](#fabric)

## <a name="overview">Overview</a>
The purpose of this quickstart is to provide a fully functional and secure Grey Matter deployment. While the deployed service mesh will be fully functional, it is not intended for production use. Further, the quickstart will not enable all features of the service mesh. For instructions on enabling and configuring more options, see the [documentation](https://docs.greymatter.io).

## <a name="dependencies">Dependencies</a>
The instructions in this quickstart assume the following dependencies are installed and available in your shell of choice. Installation and configuration of these dependencies is outside the scope of this quickstart.

1. [acert](https://github.com/deciphernow/acert)
2. [base64](https://linux.die.net/man/1/base64)
3. [git](https://git-scm.com/)
4. [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

## <a name="instructions">Instructions</a>

### <a name="setup">Setup Local Environment</a>
All intructions within this quickstart assume that you have cloned the repository locally and are working from within the `kuberenetes` directory.

1. Clone the repository locally.

    ```
    git clone https://github.com/joshua-rutherford/greymatter-tutorials.git
    ```

2. Change directories to the root of the Kubernetes tutorial.
   
    ```
    cd greymatter-tutorials/kubernetes
    ```

### <a name="spire">Deploying a SPIRE</a>
This quickstart leverages [SPIFFE](https://spiffe.io) X.509 identities issued by [SPIRE](https://spiffe.io/spire/) to uniquely identify and secure all communication within the service mesh. While not the only option available, SPIRE provides a platform agnostic solution that minimizes maintenance via automation while increasing security via key rotation and strict attestation policies. For documentation on other options that do not involve SPIRE, please see the Grey Matter [documentation](https://docs.greymatter.io). For detailed configuration options using SPIRE please see the SPIRE [documenation](https://spiffe.io/spire/docs/).

While a full overview of the deployment and configuration of SPIRE is outside the scope of this quickstart, it is important to note that the sidecars deployed are identified by the Kubernetes namespace and service account in which they run. As a result, two sidecars within a single namespace and service account are, for security purposes, identical and will recieve the same SPIFFE identity in the format:

    spiffe://<trust domain>/ns/<namespace>/sa/<service account>

For example, in this quickstart the trust domain is `quickstart.greymatter.io`, the Grey Matter Control API is deployed within the `fabric` namespace with the service account `api`.  As a result, it's SPIFFE identity is:

    spiffe://quickstart.greymatter.io/ns/fabric/sa/api

The following instructions detail the configuration and installation of the SPIRE components required by this quickstart.

1. Create a certificate authority and export the fingerprint.

    export AUTHORITY_FINGERPRINT=$(acert authorities create -n 'Quickstart' -o 'Grey Matter')

2. Create an overlay for the certificate authority secret.

    ```
    kubectl create secret generic server-ca \
      -n spire \
      -o yaml \
      --dry-run \
      --from-literal=root.crt="$(acert authorities export ${AUTHORITY_FINGERPRINT} -t authority -f pem)" \
      --from-literal=intermediate.crt="$(acert authorities export ${AUTHORITY_FINGERPRINT} -t certificate -f pem)" \
      --from-literal=intermediate.key="$(acert authorities export ${AUTHORITY_FINGERPRINT} -t key -f pem)" > \
      spire.server/overlay/server.ca.secret.yaml
    ```

3. Create a leaf certificate for the registrar webhook and export the fingerprint.

    ```
    export REGISTRAR_FINGERPRINT=$(acert authorities issue ${AUTHORITY_FINGERPRINT} -n 'registrar.spire.svc')
    ```

4. Create an overlay for the registrar secret.

    ```
    kubectl create secret generic server-tls \
      -n spire \
      -o yaml \
      --dry-run \
      --from-literal=ca.crt="$(acert leaves export ${REGISTRAR_FINGERPRINT} -t authority -f pem)" \
      --from-literal=registrar.spire.svc.crt="$(acert leaves export ${REGISTRAR_FINGERPRINT} -t certificate -f pem)" \
      --from-literal=registrar.spire.svc.key="$(acert leaves export ${REGISTRAR_FINGERPRINT} -t key -f pem)" > \
      spire.server/overlay/server.tls.secret.yaml
    ```

5. Copy the existing webhook configuration and replace the certificate authority bundle.

    ```
    sed "s/caBundle: .*/caBundle: $(acert leaves export ${REGISTRAR_FINGERPRINT} -t authority -f pem | base64)/" spire.server/base/server.validatingwebhookconfiguration.yaml > spire.server/overlay/server.validatingwebhookconfiguration.yaml
    ```

6. Deploy the Spire server using kustomize and your custom overlay.

    ```
    kubectl apply -k spire.server/overlay
    ```

7. Wait until the `server-0` pod shows a status of `Running`.

    ```
    kubectl get pods -n spire -w
    ```

8.  Stop watching the pods by entering `Ctrl+C`.

9.  Deploy the Spire agents using kustomize (no custom overlay required).

    ```
    kubectl apply -k spire.agent/base
    ```

10. Wait until all the `agent-*` pods show a status of `Running`.

    ```
    kubectl get pods -n spire -w
    ```

11. Stop watching the pods by entering `Ctrl+C`.

### <a name="fabric">Deploying a Grey Matter Fabric</a>

1. Create an overlay image pull secret for Docker Hub Grey Matter images.

    ```
    kubectl create secret docker-registry index.docker.io \
      -n fabric \
      -o yaml \
      --dry-run \
      --docker-username=${DOCKER_USERNAME} \
      --docker-password=${DOCKER_PASSWORD} > fabric/overlay/index.docker.io.secret.yaml
    ```

2. Create an overlay image pull secret for Nexus Grey Matter images.

    ```
    kubectl create secret docker-registry docker.production.deciphernow.com \
      -n fabric \
      -o yaml \
      --dry-run \
      --docker-username=${DOCKER_USERNAME} \
      --docker-password=${DOCKER_PASSWORD} > fabric/overlay/docker.production.deciphernow.com.secret.yaml
    ```

3. Deploy the Grey Matter Fabric using kustomize and your custom overly.

    ```
    kubectl apply -k fabric/overlay
    ```

4. Wait until the `api-0` and all `control-*` pods show a status of `Running`.

    ```
    kubectl get pods -n fabric -w
    ```

5. Stop watching the pods by entering `Ctrl+C`.

