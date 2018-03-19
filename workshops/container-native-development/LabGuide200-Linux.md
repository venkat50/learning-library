# Provision Kubernetes Using Terraform

![](images/200/header.png)

## Introduction

This is the second of several labs that are part of the **Oracle Public Cloud Container Native Development workshop.** This workshop will walk you through the process of moving an existing application into a containerized CI/CD pipeline and deploying it to a Kubernetes cluster in the Oracle Public Cloud.

You will take on 2 personas during the workshop. The **Lead Developer Persona** will be responsible for configuring the parts of the automated build and deploy process that involve details about the application itself. The **DevOps Engineer Persona** will configure the parts of the automation involving the Kubernetes infrastructure. To containerize and automate the building and deploying of this application you will make use of Wercker Pipelines for CI/CD, Docker Hub for a container registry, and Terraform for provisioning a Kubernetes cluster on Oracle Cloud Infrastructure.

During this lab, you will take on the **DevOps Engineer Persona**. You will provision a Kubernetes cluster and all of the infrastructure that it requires using Terraform. Terraform will provision the Virtual Cloud Network, Load Balancers, Kubernetes Master and Worker instances, and etcd instance required to support your cluster.

**_To log issues_**, click here to go to the [GitHub oracle](https://github.com/oracle/learning-library/issues/new) repository issue submission form.

## Objectives

**Automate Deployment to Kubernetes**

- Create and Deploy to a Kubernetes Cluster
  - Set Up Oracle Cloud infrastructure
  - Provision Kubernetes Using Terraform
  - Configure and Run Wercker Deployment Pipelines
  - Deploy and Test the Product Catalog Application

## Required Artifacts

- The following lab requires:
  - an Oracle Public Cloud account that will be supplied by your instructor, or a Trial Account
  - a [GitHub account](https://github.com/join)
  - a [Docker Hub account](https://hub.docker.com)

# Provision Kubernetes Using Terraform

## Set Up Oracle Cloud infrastructure

### **STEP 1**: Log in to your OCI dashboard

- From any browser, go to:

    [https://console.us-ashburn-1.oraclecloud.com/#/a/](https://console.us-ashburn-1.oraclecloud.com/#/a/)

- Enter your **Cloud Tenant** in the input field and click **Continue**. This will be supplied by your workshop instructor.

  ![](images/200/OCI-Login1.PNG)

- Enter your **Username** and **Password** in the input fields and click **Sign In**. This will be supplied by your workshop instructor (`api.user or cloud.admin`) 


  ![](images/200/OCI-Login2.PNG)

**NOTE**: If you had previously logged in, check the tenant name and just enter the username and password.

### **STEP 2**: Locate Demo Compartment for your Kubernetes nodes

Compartments are used to isolate resources within your OCI tenant. User-based access policies can be applied to manage access to compute instances and other resources within a Compartment.

- Hover over the **Identity** menu in the top navigation bar and click **Compartments**

  ![](images/200/Demo-Compartment.PNG)

- If your workshop instructor has directed you to use a pre-created compartment **_do not create a new one_**. Locate the **Demo** compartment in the list and click **Copy** next to the displayed OCID. **Paste** this OCID into a text file or elsewhere for safe keeping. We will use it to tell Terraform where to set up our cluster in a later step. Proceed to **STEP 3**.

  

### **STEP 3**: Create and upload a new API key

An API key is required for Terraform to authenticate to OCI in order to create compute instances for your Kubernetes master and worker nodes.

- Open a terminal window and run each of the following commands, one at a time, pressing **Enter** between each one. These commands will create a new directory called `.oci`, generate a new PEM private key, generate the corresponding public key, and copy the public key to the clipboard. For more information on this process, including the alternate commands to protect your key file with a passphrase, see the [official documentation](https://docs.us-phoenix-1.oraclecloud.com/Content/API/Concepts/apisigningkey.htm#two).


```bash
mkdir ~/.oci
openssl genrsa -out ~/.oci/oci_api_key.pem 2048
openssl rsa -pubout -in ~/.oci/oci_api_key.pem -out ~/.oci/oci_api_key_public.pem
cat ~/.oci/oci_api_key_public.pem
```

- Select the entire public key, beginning with `-----BEGIN PUBLIC KEY-----` and ending with `-----END PUBLIC KEY-----` and **copy** it to the clipboard.

  ![](images/200/11.png)

- In your browser window showing the OCI Console, click the **api.user** menu item and select **User Settings** as shown below:

  ![](images/200/API-User1.PNG)

- Click **Add Public Key**
  

- **Paste** the public key from your clipboard into the text field and click **Add**. Note: The public key was copied to the clipboard when you ran the `cat` command from the terminal window, which copied the results to the clipboard using the `clip` command.

  ![](images/200/13.png)

- **Leave this browser window open**, as we will need to copy and paste some of this information into the Terraform configuration file.

## Provision Kubernetes Using Terraform

### **STEP 4**: Download Terraform

- Download the appropriate Terraform package for your operating system from the [terraform.io downloads page](https://www.terraform.io/downloads.html).

  ![](images/200/58.png)

- **Unzip** the file you downloaded into the folder ~/terraform. You can use the command line or a graphical zip program for this operation. The example command below assumes you don't have other zip files in the current directory beginning with the string `terraform_`.

```bash
cd ~/Downloads
mkdir -p ~/terraform
unzip terraform_*.zip
mv terraform ~/terraform
cd ~/terraform
```

- Add Terraform to your PATH in the **terminal window** that you will _use for the next two steps_ using the following command:

```bash
export PATH=$PATH:`pwd`
```
### **STEP 5**: Download the OCI Terraform Provider

- Download the **OCI Terraform Provider** from the [GitHub release page](https://github.com/oracle/terraform-provider-oci/releases/latest). Select the package for your operating system. **Note:** for **Mac** use a **darwin** version of the tar file.

  ![](images/200/59.png)

- Click on the **Save File** option if you are prompted and then click on **OK** to download the file.

  ![](images/200/59.1.png)

- Run the following commands in a **terminal window** to extract the provider binary into the Terraform plugins folder. _Note:
  replace_ the `linux_*.tar.gz` with the filename of the file you downloaded:

  ```bash
  cd ~/Downloads
  mkdir -p ~/.terraform.d/plugins && cat linux_*.tar.gz | tar -zxvf - -C ~/.terraform.d/plugins/
  ```

- Terraform will look in the `plugins` directory for the OCI provider when it is specified by an installer, as we will see in the next step.

### **STEP 6**: Download and Configure the OCI Terraform Kubernetes Installer

- _Install kubectl_. Terraform requires `kubectl` - the Kubernetes command line interface, to interact with Kubernetes from your local machine. You can install it by following the instructions for **Installing kubcectl binary via curl** for either mac or linux **[Kubernetes docs](https://kubernetes.io/docs/tasks/tools/install-kubectl/)**.

- After you have installed kubectl, from the same **terminal window** you used in the previous step, run the following commands to download the OCI Terraform Kubernetes Installer:

  ```bash
  cd ~
  git clone https://github.com/oracle/terraform-kubernetes-installer.git
  cd terraform-kubernetes-installer
  ```

- Initialize this Terraform installer by running the following command:

  ```bash
  terraform init
  ```

- Verify proper installation of both Terraform and the OCI provider by running the following command from your **terminal window**:

  ```bash
  terraform --version
  ```
  You should see a version for `Terraform` as well as a version for the `provider.oci` plugin. It is OK if those versions differ from the screenshot below.

  ![](images/200/57.png)

- If you got the expected output from `terraform --version`, proceed to the next instruction. Otherwise, go back to the previous step and complete the instructions to **Download Terraform and the OCI Terraform Provider**. Also ensure that Terraform is in your PATH for the current terminal window.

- Once you have Terraform and the OCI provider set up, you are ready to configure the Kubernetes installer with your OCI account information. Start by making a copy of the included TFVARS example file to edit. Run the following from your **terminal window**:

  ```bash
  cp terraform.example.tfvars terraform.tfvars
  ```

- Open the `terraform.tfvars` file in your text editor of choice. On Linux you could run gedit. On a Mac, you can install gedit or use vi, but if you use TextEdit, ensure that any quotes (") you insert are not special characters:

  ```bash
  gedit terraform.tfvars
  ```

- You should still have a browser tab open to your **User Details** page in the OCI Console. If not, you can get to the User Details by selecting **Identity** then **Users** from the top menu of the Console. Then click on your **User's Name**.

- While editing the file, you will first remove the **#** comment character and replace in the values in the terraform.tfvars file on lines **2, 4, 6, and 7** using the examples in the next two images below. **NOTE**: The **region** parameter may not already be present in your tfvars file. If it is not there, add it on a new line after the user_ocid parameter on line 6.

  ![](images/200/57.1.png)


- You will replace lines **2, 4, 6, and 7** with the values from the OCI Console, referring to the following screenshot for where to find them.

  ![](images/200/17.png)

- As an exmaple, Your terraform.tfvars file should now appear similar to the image shown below:

  ![](images/200/57.2.png)

- Now follow the same process of removing the comment character **#**, and fill in the OCI Compartment ID on **line 3**. Paste the value that you saved to a text file after creating the kubernetes **compartment** in the OCI Console. If you have lost it, you can retrieve it from the OCI Console compartment list (refer to **STEP 2**).

  ```
  compartment_ocid = "Compartment OCID"
  ```

- The last piece of information we need to provide about your OCI tenant is the private key corresponding to the public API key you uploaded to the OCI console previously. Provide the path the the private key file on **line 5**. Note that _your path may differ_ from the example given below. Your public key was created as a first task in Step 3, and the location of your oci_api_key.pem file can be determined from how you completed those instructions.

  ```
  private_key_path = "/Users/your-local-username/.oci/oci_api_key.pem"
  ```

- The rest of the terraform.tfvars file controls the parameters used when creating your Kubernetes cluster. You can control how many OCPUs each node receives, whether nodes should be virtual machines or bare metal instances, how many availability domains to use, and more. We will modify three of the lines in the remainder of the file.

- First, we will specify that we want only one OCPU in each of the worker and master nodes. This reduces the hourly cost of running our cluster. On **lines 15 and 16**, uncomment the **k8sMasterShape** and **k8sWorkerShape** parameters, and set both values to **VM.Standard1.1**:

  ```
  k8sMasterShape = "VM.Standard1.1"
  k8sWorkerShape = "VM.Standard1.1"
  ```

- Next, we will specify the type of load balancers we want for the master and etcd VMs -- 400Mbps in this case. Alter **lines 30 and 31** to read:

  ```
  etcdLBShape = "400Mbps"
  k8sMasterLBShape = "400Mbps"
  ```

- The last change we will make is to open up the allowed Kubernetes master inbound IP address range, so that we can access our cluster from the internet. On **line 38**, remove the pound sign at the beginning of the line to uncomment it.

  ```
  master_https_ingress = "0.0.0.0/0"
  ```

  **NOTE**: The 0.0.0.0/0 value means that any IP address can access your cluster. A better security practice would be to determine your externally-facing IP address and restrict access to only that address. If you'd like, you can find out your IP address by running `curl ifconfig.co` in a terminal window, and place that address into the `master_https_ingress` parameter (e.g. `master_https_ingress = "11.12.13.14/32"`). Note that if you need remote assistance with the workshop, you may need to open this back up to 0.0.0.0/0 to allow access to your cluster.

- **Double check** to ensure you removed the **#** character from in front of all the entries you modified

### **STEP 7**: Provision Kubernetes on OCI

- Now we are ready to have Terraform provision our Kubernetes cluster. **Save and close** your terraform.tfvars file. In your open **terminal window**, run the following command to have Terraform evaluate the various network and compute infrastructure that we are asking to be provisioned.

  ```bash
  terraform plan
  ```

  ![](images/200/60.png)

- If the output of the plan step looks correct, you are ready to actually provision the infrastructure by running the following command in your **terminal window**. Note that Terraform will prompt you to type `yes` after it recomputes the required plan in order to begin provisioning.

  ```bash
  terraform apply
  ```

  ![](images/200/61.png)

- It will take several minutes to create the required Virtual Cloud Networks, load balancers, and compute instances that make up a Kubernetes cluster. If you'd like, you can observe the objects being created in the **OCI Console** -- click on **Compute** or **Networking** from the navigation menu and be sure to select the **kubernetes compartment** from the dropdown on the left side of the page.

  ![](images/200/62.png)

- **When provisioning is complete**, Terraform will output the details of all created infrastructure to the terminal:

  ![](images/200/63.png)

- Even though the Terraform provisioning has completed there is still configuration and setup being completed within the account. Make sure both Load Balancers are up and running before proceeding. In your account select **Networking-->Load Balancers**, and wait for the green health checkmarks to show that the Load Balances are up and running. 

  ![](images/200/63.3.png)

  ![](images/200/63.6.png)

- During provisioning, Terraform generated a `kubeconfig` file that will authenticate you to the cluster. Let's configure and start the kubectl proxy server to make sure our cluster is accessible.

- You will need to set an environment variable to point `kubectl` to the location of your Terraform-generated `kubeconfig` file. Then you can start the Kubernetes proxy server, which will let you view the cluster dashboard at a localhost URL.

  ```bash
  export KUBECONFIG=`pwd`/generated/kubeconfig
  kubectl proxy
  ```

  **NOTE**: Should you need to change the IP address of your cluster in the future, you can configure `kubectl` with the updated connection information by running the following command, which will pass the current address and authentication details to **kubectl**: `terraform output kubeconfig | tr '\n' '\0' | xargs -0 -n1 sh -c`

- With the proxy server running and the Load Balancers showing a running status, navigate to the **[Kubernetes dashboard](http://localhost:8001/api/v1/namespaces/kube-system/services/http:kubernetes-dashboard:/proxy/)** in a new browser tab.

  ![](images/200/64.png)

- Great! We've got Kubernetes installed and accessible -- now we're ready to get our microservice deployed to the cluster. The next step is to tell Wercker how and where we would like to deploy our application. In your **terminal window**, press **Control-C** to terminate `kubectl proxy`. We will need the terminal window to gather some cluster info in another step. We'll start the proxy again later.

## Configure and Run Wercker Deployment Pipelines

### **STEP 8**: Define Kubernetes Deployment Specification

- From a browser, navigate to your forked twitter-feed repository on GitHub. If you've closed the tab, you can get back by going to [GitHub](https://github.com/), scrolling down until you see the **Your repositories** box on the right side of the page, and clicking the **twitter-feed** link.

  ![](images/200/25.png)

- Click **Create new file**

  ![](images/200/27.png)

- In the **Name your file** input field, enter **kubernetes.yml.template**

  ![](images/200/28.png)

- **Copy** the YAML below and **paste** it into the file editor.

  >This configuration consists of two parts. The first section (up to line 28) defines a **Deployment**, which tells Kubernetes about the application we want to deploy. In this Deployment we instruct Kubernetes to create two Pods (`replicas: 2`) that will run our application. Within those pods, we specify that we want one Docker container to be run, and compose the link to the image for that container using environment variables specific to this workflow execution (`image: ${DOCKER_REPO}:${WERCKER_GIT_BRANCH}-${WERCKER_GIT_COMMIT}`).

  >The second part of the file defines a **Service**. A Service defines how Kubernetes should expose our application to traffic from outside the cluster. In this case, we are asking for a cluster-internal IP address to be assigned (`type: ClusterIP`). This means that our twitter feed will only be accessible from inside the cluster. This is ok, because the twitter feed will be consumed by the product catalog application that we will deploy later. We can still verify that our twitter feed is deployed properly -- we'll see how in a later step.

  >A `.yml` file is a common format for storing Kubernetes configuration data. The `.template` suffix in this file, however, is not a Kubernetes concept. We will use a Wercker step called **bash-template** to process any `.template` files in our project by substituting environment variables into the template wherever `${variables}` appear. You'll add that command to a new pipeline in the next step.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: twitter-feed-v1
  labels:
    commit: ${WERCKER_GIT_COMMIT}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: twitter-feed
  template:
    metadata:
      labels:
        app: twitter-feed
        commit: ${WERCKER_GIT_COMMIT}
    spec:
      containers:
      - name: twitter-feed
        image: ${DOCKER_REPO}:${WERCKER_GIT_BRANCH}-${WERCKER_GIT_COMMIT}
        imagePullPolicy: Always
        ports:
        - name: twitter-feed
          containerPort: ${PORT}
          protocol: TCP
      imagePullSecrets:
        - name: wercker
---
apiVersion: v1
kind: Service
metadata:
  name: twitter-feed
  labels:
    app: twitter-feed
    commit: ${WERCKER_GIT_COMMIT}
spec:
  ports:
  - port: 30000
    targetPort: ${PORT}
  selector:
    app: twitter-feed
  type: ClusterIP
---
```

- At the bottom of the page, click **Commit new file**

  ![](images/200/29.png)

- Since you've committed to the repository, Wercker will trigger another execution of your workflow. We haven't defined the deployment pipelines yet, so this will just result in a new entry in Wercker's Runs tab and a new image pushed to the container registry. You don't need to do anything with those; you can move on to the next step.

### **STEP 9**: Define Wercker Deployment Pipelines

- Click the file **wercker.yml** and then click the **pencil** button to begin editing the file.

  ![](images/200/26.png)

- **Copy** the YAML below and **paste** it below the pipelines we defined earlier.

  >This will define a new **Pipeline** called deploy-to-cluster. The pipeline will make use of a new type of step: **kubectl**. If you have used Kubernetes before, you will be familiar with kubectl, the standard command line interface for managing Kubernetes. The kubectl Wercker step can be used to execute Kubernetes commands from within a Pipeline.

  >The **deploy-to-cluster** Pipeline will prepare our kubernetes.yml file by filling in some environment variables. It will then use kubectl to tell Kubernetes to apply that configuration to our cluster.

```yaml
#Deploy our container from the Docker Hub to Kubernetes
deploy-to-cluster:
    box:
        id: alpine
        cmd: /bin/sh
    steps:

    - bash-template

    - script:
        name: "Visualise Kubernetes config"
        code: cat kubernetes.yml

    - kubectl:
        name: deploy to kubernetes
        server: $KUBERNETES_MASTER
        #username: $KUBERNETES_USERNAME
        token: $KUBERNETES_TOKEN
        insecure-skip-tls-verify: true
        command: apply -f kubernetes.yml
```

- At the bottom of the page, click **Commit new file**

  ![](images/200/29.png)

- Since you've committed to the repository again, Wercker will once again trigger an execution of your workflow. We still haven't configured the deployment pipelines in Wercker yet, so we'll still end up with a new Run and a new image, but not a deployment to Kubernetes.

### **STEP 10**: Set up deployment pipelines in Wercker

- Open **[Wercker](https://app.wercker.com)** in a new tab or browser window, or switch to it if you already have it open. In the top navigation bar, click **Pipelines**, then click on your **twitter-feed** application.

  ![](images/200/30.png)

- On the **Runs** tab you can see that Wercker has triggered another execution of our build and publish workflow, but it has not executed our new deploy-to-cluster pipeline. This is because we have not added the new pipeline to the workflow definition yet. Let's do that now -- click on the **Workflows** tab, then click the **Add new pipeline** button.

  ![](images/200/31.png)

- Enter **deploy-to-cluster** into both the Name and YML Pipleine name fields. Click **Create**.

  ![](images/200/32.png)

- Click the **Workflows** tab again to get back to the editor.

- Click the **plus** button to the right of the **push-release** pipeline to add to the workflow. In the **Execute Pipeline** drop down list, select **deploy-to-cluster** and click **Add**

  ![](images/200/33.png)

- Your overall Workflow should now have three Pipelines:

  ![](images/200/34.png)

- Now we've got our workflow updated with our deployment pipelines, but there's one more thing we need to do before we can actually deploy. We need to set two environment variables that tell Wercker the address of our Kubernetes master and provide an authentication token for Wercker to issue commands to Kubernetes.

### **STEP 11**: Set up environment variables in Wercker

- Our first step is to set our cluster's authentication token as a Wercker environment variable. In your **terminal window**, run the following command to output the token, then **select it and copy it** to your clipboard:

  ```bash
  terraform output api_server_admin_token
  ```

- Back in your Wercker browser tab, click the **Environment** tab. In the key field of the empty row below the last environment variable, enter the key **KUBERNETES_TOKEN**. In the value field, **paste** the token we just copied. Check the **Protected** box and click **Add**.

  ![](images/200/37.png)

- The other environment variable we need to add is the address of the Kubernetes master we want to deploy to. We can get the URL from `kubectl`. Run the following command in your **terminal window** to output the URL, then **select it and copy it** to your clipboard:

  ```bash
  echo $(kubectl config view | grep server | cut -f 2- -d ":" | tr -d " ")
  ```

- In your Wercker browser tab, add a new environment variable with the key **KUBERNETES_MASTER**. In the value field, **paste** the value you copied from `kubectl`. The value **must start with https://** for Wercker to communicate with the cluster. When finished, click **Add**.

  ![](images/200/55.png)

- Now we're ready to try out our workflow from start to finish. We could do that by making another commit on GitHub, since Wercker is monitoring our source code. We can also trigger a workflow execution right from Wercker. We'll see how in the next step.

### **STEP 12**: Trigger a retry of the pipeline

- On your Wercker application page in your browser, click the **Runs** tab. Your most recent run should have successful build and push-release pipelines. Click the **push-release** pipeline.

  ![](images/200/39.png)

- From the **Actions** menu, click **deploy-to-cluster**.

  ![](images/200/40.png)

- In the dialog box that appears, click **Execute pipeline**

  ![](images/200/41.png)

- Click the **Runs** tab so you can monitor the execution of the pipeline. Within a minute or so, the deployment pipeline should complete successfully. Now we can use the Kubernetes dashboard to inspect and validate our deployment.

  ![](images/200/42.png)

### **STEP 13**: Validate deployment

- In a terminal window, start the **kubectl proxy** using the following command. Your `KUBECONFIG` environment variable should still be set from a previous step. If not, reset it.

  ```bash
  kubectl proxy
  ```

- In a browser tab, navigate to the [**Kubernetes dashboard**](http://localhost:8001/api/v1/namespaces/kube-system/services/http:kubernetes-dashboard:/proxy/)

- You should see the overview page. In the pods section, you should see two twitter-feed pods running. Click the **name of one of the pods** to go to the detail page.

  ![](images/200/44.png)

- On the pod detail page, in the top menu bar, click **Exec**. This will give us a remote shell on the pod where we can verify that our application is up and running.

  ![](images/200/45.png)

- In the shell that is displayed, **paste** the following command and press **Enter**.

  **NOTE:** You may need to use ctrl-shift-v to paste. Alternatively, you can use the mouse-driven browser menu to paste the command.

  `curl -s http://$HOSTNAME:8080/statictweets | head -c 100`

- You should see some JSON data being returned by our twitter feed service. Our microservice has been deployed successfully! But the twitter feed service is just one part of our product catalog application. Let's deploy the rest of the application so we can validate that everything works together as expected. Leave this browser tab open, as we will use it in a later step.

  ![](images/200/46.png)

**NOTE**: You may be wondering why we had to use the Kubernetes remote terminal to test our application. Remember the kubernetes.yml file that we created earlier -- we specified a cluster-internal IP address for our twitter-feed service. This means that only other processes inside the cluster can reach our service. If we wanted to access our service from the internet, we could have used a load balancer instead.

## Deploy and Test the Product Catalog Application

### **STEP 14**: Download the Product Catalog Kubernetes YAML file

- From a browser, navigate to your forked twitter-feed repository on GitHub. If you've closed the tab, you can get back by going to [GitHub](https://github.com/), scrolling down until you see the **Your repositories** box on the right side of the page, and clicking the **twitter-feed** link.

  ![](images/200/25.png)

- Click on the **alpha-office-product-catalog.kubernetes.yml** file.

  ![](images/200/47.png)

- Right click on the **Raw** button and choose **Save Link As** or **Save As**. In the save file dialog box that appears, note the location of the file and click **Save**

  ![](images/200/48.png)

**NOTE**: This YAML file contains the configuration for a Kubernetes deployment and service, much like the configuration for our twitter feed microservice. In a normal development environment, the product catalog application would be managed by Wercker as well, so that builds and deploys would be automated. In this workshop, however, you will perform a one-off deployment of a pre-built Docker image containing the product catalog application from within the Kubernetes dashboard.

### **STEP 15**: Deploy and test the Product Catalog using the Kubernetes dashboard

- Switch back to your **Kubernetes dashboard** browser tab. If you have closed it, navigate to the Kubernetes dashboard at [**Kubernetes dashboard**](http://localhost:8001/api/v1/namespaces/kube-system/services/http:kubernetes-dashboard:/proxy/)

- In the upper right corner of the dashboard, click **Create**.

  ![](images/200/49.png)

- Click the **Upload a YAML or JSON file** radio button, then click the **three dots** button to browse for your file. In the dialog, select the YAML file you just downloaded from GitHub and click **Open**, then click **UPLOAD**.

  ![](images/200/50.png)

- In the left side navigation menu, click **Overview**. You should see two new product-catalog-app pods being created and soon change state to Running.

  ![](images/200/51.png)

- Instead of a cluster-internal IP address, the product-catalog-service will be exposed to the internet via a load balancer. The load balancer will take a few minutes to be instantiated and configured. Let's check on its status--click **Services** from the left side menu, then click on the **product-catalog-service**.

  ![](images/200/52.png)

- On the service detail page, you will see a field called **External endpoints**. Once the load balancer has finished provisioning, the External endpoints field will be populated with a link to the product catalog application. If the link is not shown yet, wait a few minutes, refresh your browser, and check again. Once the link is displayed, **click it** to launch the site in a new tab.

  ![](images/200/53.png)

- You should see the product catalog site load successfully, validating that our new Kubernetes deployment and service were created correctly. Let's test the twitter feed functionality of the catalog. Click the first product, **Crayola New Markers**. The product's twitter feed should be displayed.

  ![](images/200/54.png)

  **NOTE**: You may have noticed that we did not need to alter the pre-built product catalog container with the URLs of the twitter feed pods or service. The product catalog app makes use of Kubernetes DNS to resolve the service name (twitter-feed) into its IP address. Kubernetes DNS assigns a DNS name to every service defined in your cluster, so any service can be looked up by doing a DNS query for the name of the service (prefixed by _`namespace.`_ if the service is in a different namespace from the requester). The product catalog server uses the following JavaScript code to make an HTTP request to the twitter feed microservice:

  `request('http://twitter-feed:30000/statictweets/color', function (error, response, body) { ... });`

- Some tweets are indeed displayed, but they aren't relevant to this product. It looks like there is a bug in our twitter feed microservice! Continue on to the next lab to explore how to make bug fixes and updates to our microservice.

**You are now ready to move to the next lab.**
