# A Jupyter Operator (for the Data Manager API)

[![build](https://github.com/informaticsmatters/data-manager-jupyter-operator/actions/workflows/build.yaml/badge.svg)](https://github.com/informaticsmatters/data-manager-jupyter-operator/actions/workflows/build.yaml)
[![build latest](https://github.com/informaticsmatters/data-manager-jupyter-operator/actions/workflows/build-latest.yaml/badge.svg)](https://github.com/informaticsmatters/data-manager-jupyter-operator/actions/workflows/build-latest.yaml)
[![build tag](https://github.com/informaticsmatters/data-manager-jupyter-operator/actions/workflows/build-tag.yaml/badge.svg)](https://github.com/informaticsmatters/data-manager-jupyter-operator/actions/workflows/build-tag.yaml)
[![build stable](https://github.com/informaticsmatters/data-manager-jupyter-operator/actions/workflows/build-stable.yaml/badge.svg)](https://github.com/informaticsmatters/data-manager-jupyter-operator/actions/workflows/build-stable.yaml)

![GitHub](https://img.shields.io/github/license/informaticsmatters/data-manager-jupyter-operator)

![GitHub tag (latest SemVer pre-release)](https://img.shields.io/github/v/tag/informaticsmatters/data-manager-jupyter-operator?include_prereleases)

This repo contains a Kubernetes _Operator_ based on the [kopf] and [kubernetes]
Python packages that is used by the **Informatics Matters Data Manager API**
to create Jupyter Notebooks for the Data Manager service.

The operator's Custom Resource Definition (CRD) can be found in
`roles/operator/files`.

By default, the operator creates instances using the Jupyter image: -

-   `jupyter/minimal-notebook:notebook-6.3.0` (see `handlers.py`)

Prerequisites: -

-   Python (ideally 3.9.x)
-   docker-compose
-   A kubernetes config file

## Building the operator
The operator container, residing in the `operator` directory,
is automatically built and pushed to Docker Hub using GitHub Actions.

You can build the image yourself using docker-compose.
The following will build an operator image with the tag `1.0.0-alpha.1`: -

    $ export IMAGE_TAG=1.0.0-alpha.1
    $ docker-compose build

## Deploying into the Data Manager API
We use [Ansible] 3 and community modules in [Ansible Galaxy] as the deployment
mechanism, using the `operator` Ansible role in this repository and a
Kubernetes config (KUBECONFIG). All of this is done via a suitable Python
environment using the requirements in the root of the project...

    $ python -m venv ~/.venv/data-manager-jupyter-operator
    $ source ~/.venv/data-manager-jupyter-operator/bin/activate
    $ pip install --upgrade pip
    $ pip install -r requirements.txt
    $ ansible-galaxy install -r requirements.yaml

Now, create a parameter file (i.e. `parameters.yaml`) based on the project's
`example-parameters.yaml`, setting values for the operator that match your
needs. Then deploy, using Ansible, from the root of the project: -

    $ export PARAMS=parameters
    $ ansible-playbook -e @${PARAMS}.yaml site.yaml

To remove the operator (assuming there are no operator-derived instances)...

    $ ansible-playbook -e @${PARAMS}.yaml -e jo_state=absent site.yaml

>   The current Data Manager API assumes that once an Application (operator)
    has been installed it is not removed. So, removing the operator here
    is described simply to illustrate a 'clean-up' - you would not
    normally remove an Application operator in a production environment.

### Deploying to the official cluster
The parameters used to deploy the operator to our 'official' cluster
are held in this repository as Ansible vault files. They can be used and edited
in-place.

Choose the _staging_ or _production_ deployment, then, to view or edit: -

    $ export PARAMS=staging
    $ ansible-vault edit ${PARAMS}-parameters.yaml.vault
    Vault password: [...]

To deploy: -

    $ ansible-playbook --ask-vault-pass \
        -e @${PARAMS}-parameters.yaml.vault site.yaml

>   You will need the vault password, held in the company's KeePass under
    `data-manager-jupyter-operator -> Ansible Vault Password`

# Data Manager Application Compliance
In order to expose the CRD as an _Application_ in the Data Manager API service
you will need to a) annotate the CRD and b) provide a **Role** and
**RoleBinding**.

## Custom Resource Definition (CRD) annotations
For the **CRD** to be recognised by the Data Manager API it wil need a number of
annotations, located in its `metadata -> annotations` block.
You will need: -

-   An annotation `data-manager.informaticsmatters.com/application` set to `'yes'`
-   An annotation `data-manager.informaticsmatters.com/application-url-location`.
    The url location is the 'status'-relative path in the custom resource
    'status' block where the application URL can be extracted. A value of
    `jupyter.notebook.url` would imply that the Application URL
    can be found in the custom resource object using the Python dictionary
    reference: `custom_resource['status']['jupyter']['notebook']['url']`.

>   Our CRD already contains suitable annotations
    (see `roles/operator/files/crd.yaml`), so there's nothing more to
    do here once you've deployed it (using Ansible in our case,
    as described earlier).

## Pod labels
So that **Pod** instances can be recognised by the Data Manager API the
application's **Pod** (only one if there are many) must contain the following
label: -

    data-manager.informaticsmatters.com/instance

Which must have a value that matched the `name` given to the operator
by the Data Manager. The name is a unique reference for the application
instance.

>   See the `spec.template.metadata.labels` block in the `deployment_body`
    section of the `create()` function in our `operator/handlers.py`.

## Role and RoleBinding definitions
As well as providing RBAC for the Operator you will need a **Role** and
**RoleBinding** to allow the Data Manager to execute the Operator. These must
allow the Data Manager to launch instances of the Custom Resource in the
Data Manager's **Namespace**.

Typical **Role** and **RoleBinding** definitions are provided in this
repository. Once you define yours you'll just need to create them: -

    $ kubectl create -f data-manager-rbac.yaml

With this done the application should be visible through the Data Manager API's
**/application** REST endpoint.

## Security context
The Custom Resource must expose properties that allow a custom
**SecurityContext** to be applied. If not, the application instance will not be
able to access the Data Manager Project files. The Data-Manager API will
expect to provide the following properties through the **CRD** schema's: -

-   `spec.deployment.securityContext.runAsUser`
-   `spec.deployment.securityContext.runAsGroup`

To run successfully the container must be able to run without privileges
and run using a user and group that is assigned by the Data Manager API.

>   See our handling of these values in the `create()` function
    of our `operator/handlers.py` and their definitions
    in `roles/operator/files/crd.yaml`

## Storage volume
In order to place Data-Manager Project files the **CRD** must
expose the following properties through its schema's: -

-   `spec.storage.claimName`
-   `spec.storage.subPath`

These will be expected to provide a suitable volume mount within the
application **Pod** for the Project files.

>   See our use of these values in `roles/operator/files/crd.yaml`.

## Instance certificate variables
Applications can use the DM-API ingress, if they use path-based routing,
and are happy to share the DM-API domain. Doing this means you won't need
a separate TLS certificate, instead using the Data Manager's.

The Jupyter operator supports this vis a Pod environment variable that is
set if you provide a value for the Ansible playbook variable
`jo_ingress_tls_secret`. If left blank the operator will expect to use the
Kubernetes [Certificate Manager], where you are expected to provide the
certificate issuer name using the playbook variable `jo_ingress_cert_issuer`.

Both are exposed in the example parameter file `example-parameters.yaml`.

---

[ansible]: https://www.ansible.com
[ansible galaxy]: https://galaxy.ansible.com
[certificate manager]: https://cert-manager.io/docs/installation/kubernetes/
[kopf]: https://pypi.org/project/kopf/
[kubernetes]: https://pypi.org/project/kubernetes/
