# Start your first Pulumi stack and deploy infrastructure

## Introduction

In this lab you will create your first Pulumi stack from a pre-built YAML template. The stack will deploy the open source project Backstage on [OCI Container Instances](https://www.oracle.com/cloud/cloud-native/container-instances/).

Estimated time: 20 minutes

### Objectives

* Set up and run your first Pulumi stack
* Access the Backstage portal
* Create a new environment in Pulumi

## Task 1: Download the code and confnigure Pulumi

1. Within your lab environment (Compute instance or local machine), clone the source code repository.

    ```bash
    <copy>
    git clone https://github.com/enschilling/oci-pulumi-self-service.git
    cd oci-pulumi-self-service/livelabs/ocw24-livelabs/99-resources/00-backstage/
    </copy>
    ```

2. Confirm Pulumi installed successfully.

    ```bash
    <copy>pulumi --help</copy>
    ```

    **Output should look something like...**

    ```bash
    Pulumi - Modern Infrastructure as Code

    To begin working with Pulumi, run the `pulumi new` command:

        $ pulumi new

    This will prompt you to create a new project for your cloud and language of choice.

    The most common commands from there are:

        - pulumi up       : Deploy code and/or resource changes
        - pulumi stack    : Manage instances of your project
        - pulumi config   : Alter your stack"'"s configuration or secrets
        - pulumi destroy  : Tear down your stack"'"s resources entirely

    For more information, please visit the project page: https://www.pulumi.com/docs/
    ```

## Create a Python virtual environment and install the Pulumi OCI provider

1. Create a new Python virtual environment with Python 3.10

    ```bash
    <copy>
    python3.10 -m venv venv
    source venv/bin/activate
    <copy>
    ```

    You should now see `(venv)` at the beginning of of your terminal prompt.

2. Install the Pulumi OCI provider

    ```bash
    <copy>
    pip install pulumi_oci
    </copy>
    ```

3. Create your first Pulumi stack.

    ```bash
    <copy>
    pulumi config
    </copy>
    ```

    Whem prompted to log in, paste your Pulumi Personal Access Token that you created in lab 1. You will then be prompted:

    ```bash
    (venv) ubuntu@instance:~/oci-pulumi-self-service/livelabs/ocw24-livelabs/99-resources/00-backstage$ pulumi config
    Manage your Pulumi stacks by logging in.
    Run `pulumi login --help` for alternative login options.
    Enter your access token from https://app.pulumi.com/account/tokens
        or hit <ENTER> to log in using your browser                   : *********************************


    Welcome to Pulumi!

    Pulumi helps you create, deploy, and manage infrastructure on any cloud using
    your favorite language. You can get started today with Pulumi at:

        https://www.pulumi.com/docs/get-started/

    Tip: Resources you create with Pulumi are given unique names (a randomly
    generated suffix) by default. To learn more about auto-naming or customizing resource
    names see https://www.pulumi.com/docs/intro/concepts/resources/#autonaming.


    Please choose a stack, or create a new one:  [Use arrows to move, type to filter]
    > <create a new stack>
    ```

4. Provide a name the stack and press enter.

## Task 2: Build out your Pulumi project

1. Now that you have a Pulumi stack, you'll need to provide some environment configuration details. Copy the commands below to a text file and replace the `<placeholder>` values with the data you gathered in lab 1.

    ```bash
    <copy>
    pulumi config set oci:region <chosen region>
    pulumi config set username <your usernam> # for the Oracle Container Registry
    pulumi config set auth-token <your auth token> --secret # for the Oracle Container Registry
    pulumi config set github-token <your github PAT> --secret 
    pulumi config set pulumi-pat <your Pulumi personal access token> --secret
    pulumi config set compartment_ocid <your compartment OCID>
    pulumi config set tenancy_ocid <your tenancy OCID>
    </copy>
    ```

2. Double check to ensure all the values were stored properly.

    ```bash
    <copy>pulumi config</copy>
    ```

    Output should look like this. Notice how the value of the secrets is not displayed:

    ```bash
    KEY               VALUE
    auth-token        [secret]
    pulumi-pat        [secret]
    github-token      [secret]
    compartment_ocid  ocid1.compartment.oc1..aa00000000000000000000000000000000000000000000006a
    tenancy_ocid      ocid1.tenancy.oc1.aa999999999999999999999999999999999999999999999b
    username          el123456g@domain.com
    oci:region        us-ashburn-1

3. Bring the Pulumi project online.

    ```bash
    <copy>pulumi up</copy>
    ```

4. 

## Task 3: Get to know your OKE cluster

Now that Kubernetes is up and running, interacting with the cluster is pretty much the same as if you were running Kubernetes on your own equipment. Let's take a look at some of the resources that get created as part of the initial setup.

1. First and foremost, what do we know about our new OKE cluster? Type `kubectl cluster-info` to find out!

2. Take a look at the default set of namespaces with `kubectl get namespaces`. You should see *default*, *kube-node-lease*, *kube-public*, and *kube-system*.

3. Check to see which pods are running across all namespaces with `kubectl get pods -o wide -A`. You should see the likes of coredns, flannel, kube-proxy, and more.

4. Are there any ingress resources defined? Try `kubectl get ingress -A` - the results should be empty as we've not yet deployed an ingress controller or defined any ingress resources.  

5. Minimize (but do not exit) Cloud Shell.

## Task 4: Create a Container Registry Repo

The OCI Container Registry (OCIR) is a secure, Dockerhub-compliant service that enables you to store and manage your container iamges securely, within the confines of OCI. 

<details><summary><b>Prefer to work with the CLI?</b></summary>

The instructions below will take you through creating a new repo via the Web UI. If you'd prefer to create the repo using the OCI CLI, you may remain in Cloud Shell and run this command (make sure to adjust the parameter value to reflect your own compartment OCID).

    ```bash
    <copy>
	oci artifacts container repository create --compartment-id ocid1.compartment.oc1..aaaaaaaace...... --display-name okeapprepo
    </copy>
    ```

---
</details>

1. Navigate to **`Developer Services`** -> **`Container Registry`**

2. Click **Create repository**

3. Provide a name for your repo and ensure it is set to **Private** access.

    ![Create new repo](images/create-repo.png)

4. Click **`[Create]`**

## Task 5: Register a secret in Kubernetes

1. First things first - you'll need to locate the region key for your selected region. This will be used to connect to the apprpriate Container Registry Endpoint. Return to Cloud Shell and enter the following command:

    ```bash
    <copy>
    oci iam region list --query 'data[?name == `us-phoenix-1`].key'
    </copy>
    ```

    >NOTE: if not using Phoenix, replace the region name with that which you've selected for the workshop.

2. Your Container Register endpoint will thus be the **key** plus `.ocir.io`. *i.e.* `phx.ocir.io`

3. For this next command you'll need to retrieve your Auth token which was created in the first lab. Construct the following command, making sure to input your own details:

    ```bash
    <copy>
    kubectl create secret docker-registry ocirsecret --docker-server='container registry endpoint' --docker-username='complete username' --docker-password='auth token' --docker-email='your email address'
    </copy>
    ```
    
    * container registry endpoint = i.e. phx.ocir.io
    * complete username = `<tenancy namespace>/<username or email address>`
        *i.e. abc123dev456/eli.schilling@oracle.com*
    * auth token = the value of the token created in lab 1

4. Validate that the secret was created successfully:

    ```
    <copy>
    kubectl get secrets
    <copy>
    ```

    ```
    user123@cloudshell:~ (us-phoenix-1)$ kubectl get secrets
    NAME         TYPE                             DATA   AGE
    ocirsecret   kubernetes.io/dockerconfigjson   1       3m
    ```


You may now **proceed to the next lab**.

## Learn More

* [Oracle Container Engine for Kubernetes (OKE)](https://www.oracle.com/cloud/cloud-native/container-engine-kubernetes/)


## Acknowledgements

* **Author** - 
* **Contributors** -
* **Last Updated By/Date** -