- [**Introduction**](#Introduction)
  - [EC2 instance / Local machine](#EC2-instance--Local-machine)
  - [EKS](#EKS)
  - [RDS](#RDS)
- [**Setup Airflow on Your Local Machine**](#Setup-Airflow-on-Your-Local-Machine)
  - [Installation](#Installation)
  - [Congifuration](#Congifuration)
- [**Setup Kubernetes on AWS EKS**](#Setup-Kubernetes-on-AWS-EKS)
  - [Prerequisite](#Prerequisite)
  - [Create EKS Cluster](#Create-EKS-Cluster)
  - [Create a kubeconfig file](#Create-a-kubeconfig-file)
  - [Launch and Configure Amazon EKS Worker Nodes](#Launch-and-Configure-Amazon-EKS-Worker-Nodes)
- [**Deploy the Kubernetes Web UI (Dashboard)**](#Deploy-the-Kubernetes-Web-UI-Dashboard)
  - [Deploy dashboard](#Deploy-dashboard)
  - [Create an eks-admin Service Account and Cluster Role Binding](#Create-an-eks-admin-Service-Account-and-Cluster-Role-Binding)
  - [Connect to the Dashboard](#Connect-to-the-Dashboard)
- [**Setup Helm**](#Setup-Helm)
  - [Installation](#Installation-1)
  - [Initialization](#Initialization)
  - [Uninstallation](#Uninstallation)
- [**Deploy Data Processing Pipeline Image on EKS**](#Deploy-Data-Processing-Pipeline-Image-on-EKS)
  - [deployment on EKS](#deployment-on-EKS)
- [**Post Configuration for Airflow**](#Post-Configuration-for-Airflow)
- [**Summary**](#Summary)

Copyright (c) 2019 Bosch

Last modified date: Monday, July 1st 2019, 1:51:13 pm

# **Introduction**

This is a tutorial that guides you through the basic concepts and steps of deploying Kubernetes on AWS EKS with Dask distributed support, and control EKS kubernetes cluster from your local machine.

## EC2 instance / Local machine

- Libraries

    **Airflow:** Airflow is a platform to programmatically author, schedule and monitor workflows.Use airflow to author workflows as directed acyclic graphs (DAGs) of tasks. The airflow scheduler executes your tasks on an array of workers while following the specified dependencies. [Reference](https://airflow.apache.org/)

    **Kubectl:** The Kubernetes command-line tool, kubectl, allows you to run commands against Kubernetes clusters. You can use kubectl to deploy applications, inspect and manage cluster resources, and view logs. [Reference](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

    **Helm:** Helm helps you manage Kubernetes applications — Helm Charts help you define, install, and upgrade even the most complex Kubernetes application. [Reference](https://helm.sh/)

## EKS

- Service

    **Kubernetes:**
    Kubernetes (K8s) is running as a service on AWS EKS, it is the core service on EKS. Note: EKS stands for Elastic Container Service for Kubernetes, not equals to Kubernetes. Kubernetes provides automated container deployment, scaling and management.[Reference](https://kubernetes.io/)

    **Dask:**
    Dask is a Python library that integrates with existing Python projects, like Pandas, Numpy and Scikit-Learn, that can distribute multi-dimensional data computation across multiple workers so that speed up data processing.[Reference](https://dask.org/)

- System

    **Kubernetes-dashboard:**
    Dashboard is a web-based Kubernetes user interface. You can use Dashboard to deploy containerized applications to a Kubernetes cluster, troubleshoot your containerized application, and manage the cluster resources. You can use Dashboard to get an overview of applications running on your cluster, as well as for creating or modifying individual Kubernetes resources (such as Deployments, Jobs, DaemonSets, etc). For example, you can scale a Deployment, initiate a rolling update, restart a pod or deploy new applications using a deploy wizard.
    [Reference](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)

    **Heapster:**
    Heapster enables Container Cluster Monitoring and Performance Analysis for Kubernetes (versions v1.0.6 and higher), and platforms which include it. Note: Heapster is now retired, for the future we will replace it with other Kubernetes Metrics.[Reference](https://github.com/kubernetes-retired/heapster)

    **Influxdb:**
    Influxdb is used for Heapster backend.[Reference](https://github.com/kubernetes-retired/heapster/blob/master/docs/influxdb.md)

    **Tiller:**
    Tiller is the in-cluster component of Helm. It interacts directly with the Kubernetes API server to install, upgrade, query, and remove Kubernetes resources. It also stores the objects that represent releases.[Refernce](https://helm.sh/docs/glossary/#tiller)

## RDS

- **PostgreSQL database:**
    PostgreSQL is a powerful, open source object-relational database system with over 30 years of active development that has earned it a strong reputation for reliability, feature robustness, and performance. There we use PostgreSQL as airflow metadata backend database, airflow scheduler and all the workers will access it to sync dag status. [Reference](https://www.postgresql.org/)

# **Setup Airflow on Your Local Machine**

## Installation

- Using this command line to install Airflow.
  
    ```bash
    pip install apache-airflow[postgres]
    ```

  - Verify airflow installation using following command.

    ```bash
    airflow version # it's recommended to run it, cause it will create airflow.cfg by default
    ```

## Congifuration

- Modify source code in airflow library. When using dask executor in Airflow, there is a aready known bug exsit, please modify the file `dask_executor.py` in your airflow library to avoid the issue.

    ```python
    # file location:
    # PATH/TO/PYTHON/LIBRARY/airflow/executors/dask_executor.py
    # e.g.
    #   ~/venv/lib/python3.6/site-packages/airflow/executors/dask_executor.py

    # locate line #68 and find
    shell=True
    # set it to
    shell=False
    ```

- Configure `~/airflow/airflow.cfg`

    ```ini
    #
    # 1. locate [core] section and find the variable 'dags_folder', it tells \
    #    airflow the directory to find dags. For the consistency between cluster \
    #    and local machine, please set it to the following values to match the \
    #    same dags folder directory in cloud cluster.
    #
    dags_folder = /root/airflow/dags

    #
    # 2. locate [core] section and find the variable 'sql_alchemy_conn', \
    #    it defines the backend where airflow metadata stores.
    #
    #    $DATABASE_USERNAME: user name of postgresql database created on aws \
    #    $DATABASE_PASSWORD: password for the postgresql database
    #    $POSTGRESQL_ADDRESS: postgresql database address, it should
    #        look like DATABASE_ID.abcdefg123.REGION.rds.amazonaws.com:5432
    #    $DATABSE_NAME: name of postgresql database
    #
    #    E.g.
    #    sql_alchemy_conn = "postgresql+psycopg2://username:password@pipeline-postgresql.abcdefghijk.us-east-2.rds.amazonaws.com:5432/pipeline_postgresql"
    #
    sql_alchemy_conn = "postgresql+psycopg2://$DATABASE_USERNAME:$DATABASE_PASSWORD@$POSTGRESQL_ADDRESS
    :5432/$DATABSE_NAME"

    #
    # 3. locate [core] section and find the variable 'executor', it assigns \
    #    Dask scheduler as the airflow task scheduler.
    #
    executor = DaskExecutor

    #
    # 4. locate [core] section and find the variable 'sql_alchemy_pool_enabled', \
    #    it indicates airflow not to use alchemy pool, \
    #    it will cause unknow error when using dask executor.
    #
    sql_alchemy_pool_enabled = False
    ```

# **Setup Kubernetes on AWS EKS**

[Reference](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html)

## Prerequisite

  1. AWS account with following policies: _(If you are admin user, you can skip this part)_
      - Full access to RDS
      - IAM: Role
      - IAM: InstanceProfile
      - EC2: SecurityGroup
      - EC2: SecurityGroupIngress
      - EC2: SecurityGroupEgress
      - EC2: InternetGateway
      - EC2: Route
      - EC2: RouteTable
      - EC2: Subnet
      - EC2: SubnetRouteTableAssociation
      - EC2: VPC
      - EC2: VPCGatewayAttachment
      - AutoScaling: LaunchConfiguration
      - AutoScaling: AutoScalingGroup

  2. Create EKS service role for this AWS account:

      - `IAM` -> `Roles` -> `Create role`\
      <img src="image/create-eks-service-role-1.png" width="800" height="300" />

      - -> `EKS` -> `Next: permission`\
      <img src="image/create-eks-service-role-2.png" width="550" height="300" />

      - -> `Next: tag` -> `Next: review` -> `provide a role name: $ROLE_NAME` -> `Create role`\
      <img src="image/create-eks-service-role-3.png" width="550" height="300" />
        - $ROLE_NAME: eksServiceRole _(just recommended)_

  3. <a id="Prerequisite-3"></a>Create EKS cluster VPC using `CloudFormation` with VPC template

      - `CloudFormation` -> `Stacks details` -> `Create stack`\
      <img src="image/create-eks-vpc-1.png" width="300" height="150" />

      - -> `Amazon S3 URL: $TEMPLATE_URL` -> `Next`\
      <img src="image/create-eks-vpc-2.png" width="550" height="300" />
        - $TEMPLATE_URL: <https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-02-11/amazon-eks-vpc-sample.yaml>

      - -> `stack name: $STACK_NAME` -> `Next to Create stack`\
      <img src="image/create-eks-vpc-3.png" width="550" height="300" />
        - $STACK_NAME: eks-vpc _(just recommended)_

      - Check the output of this stack, please keep the `SecurityGroup` and `Vpcid` in mind, we will use that later:\
      <img src="image/eks-vpc-stack-output.png" width="600" height="200" />

  4. Install and configure `kubectl` for EKS, Kubernetes uses a command-line utility called kubectl for communicating with the cluster API server. [Install Tutorial](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)

  5. Install and configure the latest `AWS CLI`. [Install Tutorial](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)
      - After installation, using this command to configure your aws credential:

        ```bash
        aws configure
        ```

        Then the AWS CLI will prompt you to enter following credential:

        ```bash
        AWS Access Key ID [None]: #YOUR_ACCESS_KEY
        AWS Secret Access Key [None]: #YOUR_SECRET_KEY
        Default region name [None]: us-east-2
        Default output format [None]: json
        ```

        The aws credential will be stored in a profile, you can find in this directory `~/.aws/` based on Unix System.

  6. Install `aws-iam-authenticator` and authoturize aws to kubectl, Amazon EKS uses IAM to provide authentication to your Kubernetes cluster through the AWS IAM Authenticator for Kubernetes. [Install Tutorial](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html)

## Create EKS Cluster

   - `EKS` -> `Create cluster`\
   <img src="image/create-eks-cluster-1.png" width="800" height="250"/>

   - -> `Cluster name: $CLUSTER_NAME` -> `Role name: $ROLE_NAME` -> `VPC: $VPC` -> `Subnets: $SUBNETS`\
   <img src="image/create-eks-cluster-2.png" width="800" height="450"/>
     - $CLUSTER_NAME: eks-cluster _(just recommended)_
     - $ROLE_NAME: eksServiceRole
     - $VPC: eks-vpc-VPC 
     - $SUBNETS: subnets in eks-vpc-VPC

   - -> `Security groups: $SECURITY_GROUP` -> `Create`\
   <img src="image/create-eks-cluster-3.png" width="800" height="450"/>
     - $SECURITY_GROUP: Security group for the cluster control plane communication with worker nodes, which you just created in <a href="#Prerequisite-3">create vpc</a>.

## Create a kubeconfig file

  1. Create a kubeconfig file using follow command.

      ```bash
      aws eks --region $REGION update-kubeconfig --name $CLUSTER_NAME 
      ```

      - $REGION: us-east-2
      - $CLUSTER_NAME: eks-cluster

      ```bash
      kubectl get service
      # use this cmd to test connection, by default
      # the output only has one service: "kubernetes"
      ```

## Launch and Configure Amazon EKS Worker Nodes

  1. Launch and Configure Amazon EKS Worker Nodes (nodegroup) using CloudFormation.     -> `stack name: $STACK_NAME` -> `Next to Create stack`

      - `CloudFormation` -> `Stacks details` -> `Create stack`\
      <img src="image/create-eks-vpc-1.png" width="300" height="150"/>

      - `Amazon S3 URL: $TEMPLATE_URL`\
      <img src="image/create-nodegroup-1.png" width="500" height="250"/>
        - $TEMPLATE_URL: <https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-02-11/amazon-eks-nodegroup.yaml>

      - -> `Stack name: STACK_NAME` -> `EKS ClusterName: $CLUSTER_NAME` -> `ClusterControlPlaneSecurityGroup: $SECURITY_GROUP` -> `NodeGroupName: $NODE_GROUP_NAME`\
      <img src="image/create-nodegroup-2.png" width="500" height="250"/>
        - $STACK_NAME: eks-cluster-worker-nodes _(just recommended)_
        - $CLUSTER_NAME: eks-cluster _(just recommended)_
        - $SECURITY_GROUP: Security group for the cluster control plane communication with worker nodes, which you just created in <a href="#Prerequisite-3">create vpc</a>.
        - $NODE_GROUP_NAME: eks-nodegroup _(just recommended)_

      - -> `NodeImageId: $IMG_ID` -> `KeyName: $KEY_NAME` -> `VpcId: $VPC` -> `Subnets: $SUBNETS` -> `Next`\
      <img src="image/create-nodegroup-3.png" width="500" height="250"/>
        - $IMG_ID: ami-04ea7cb66af82ae4a
        - $KEY_NAME: an EC2 key pair create by yourself
        - $VPC: eks-vpc-VPC
        - $SUBNETS: subnets in eks-vpc-VPC

  2. To enable worker nodes to join your cluster

      - Download the configuration map with the following command:

        ```bash
        curl -o aws-auth-cm.yaml https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-02-11/aws-auth-cm.yaml
        ```

      - Open the file and replace the `<ARN of instance role (not instance profile)>` snippet with the NodeInstanceRole value that you recorded in the previous procedure, and save the file.

      - Apply the configuration. This command might take a few minutes to finish.

        ```bash
        kubectl apply -f aws-auth-cm.yaml
        ```

# **Deploy the Kubernetes Web UI (Dashboard)**

[Reference](https://docs.aws.amazon.com/eks/latest/userguide/dashboard-tutorial.html)

## Deploy dashboard

- deploy dashboard

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
   ```

- deploy heapster

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml
   ```
 
- deploy influxdb backend for heapster

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml
   ```

- Create the heapster cluster role binding for the dashboard\

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml
   ```

## Create an eks-admin Service Account and Cluster Role Binding
  
- create `eks-admin-service-account.yaml` with these content

    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
     name: eks-admin
     namespace: kube-system
    ---
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRoleBinding
    metadata:
     name: eks-admin
    roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: cluster-admin
    subjects:
    - kind: ServiceAccount
     name: eks-admin
     namespace: kube-system
    ```

    Then run this command:\
    `kubectl apply -f eks-admin-service-account.yaml`

## Connect to the Dashboard

- Retrieve an authentication token for the eks-admin service account:

    ```bash
    kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
    ```
  
- Start the kubectl proxy:

    ```bash
    kubectl proxy
    ```
  
  - Open web browser to access dashboard endpoint using this [link](http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login).

  - Use the authentication token ouput from previous command to login dashboad

# **Setup Helm**

[Reference](https://zero-to-jupyterhub.readthedocs.io/en/v0.4-doc/setup-helm.html)

## Installation

- The simplest way to install helm is to run Helm’s installer script at a terminal:

    ```bash
    curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash
    ```

## Initialization

- After installing helm on your machine, initialize helm on your Kubernetes cluster. At the terminal, enter:

    ```bash
    kubectl --namespace kube-system create sa tiller

    kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller

    helm init --service-account tiller
    ```
  
- Verify the Helm installation

    ```bash
    helm version
    ```

    The output should look like:

    ```bash
    Client: &version.Version{...}
    Server: &version.Version{...}
    ```

## Uninstallation

- In case of failure of tiller pods/deployment, we can use following commands to uninstall tiller on server:

    ```bash
    kubectl -n kube-system delete deployment tiller-deploy

    kubectl delete clusterrolebinding tiller

    kubectl -n kube-system delete serviceaccount tiller
    ```

# **Deploy Data Processing Pipeline Image on EKS**

In this tutorial section, we assume that you aready have `pipeline-chart` helm chart.

## deployment on EKS

  1. Using following commands to install pipeline on Kubernetes cluster, `pipeline-chart` is the directory which contains your pipeline helm chart.

      ```bash
      helm install pipeline-chart/ --name pipeline
      ```

  2. Using following commands to uninstall pipeline on Kubernetes cluster

      ```bash
      helm del --purge pipeline
      ```

# **Post Configuration for Airflow**

- More Configuration in `~/airflow/airflow.cfg`

    ```ini
    #
    # 1. locate [dask] section 'cluster_address', it defines \
    #    the dask cluster address to find, the value should \
    #    be set once we deploy our data processing pipeline on \
    #    EKS, please set it to the dask-scheduler endpoint address.
    #
    # NOTE: you can obtain de dask-scheduler endpoint address by \
    #       using the following command:
    #
    #       `kubectl get svc`
    #
    #       you can find the pipeline-scheduler EXTERNAL_IP address, \
    #       that is the dask-scheduler endpoint address.
    #
    cluster_address = $(dask-scheduler endpoint address)
    ```

- Start Airflow Scheduler and Airflow Webserver. Once you complete the EKS setup and data preprocessing pipeline deployment, you then can use the following command to start the airflow GUI to manage and monitor your pipeline.

    ```bash
    # to start airflow scheduler
    airflow scheduler
    # to start airflow webserver at port 8080
    airflow webserver
    ```

# **Summary**

Now you have deployed dask & airflow on cloud kubernetes cluster, and runing airflow on your local machine that can remotely control kubernetes cluster. In this summary, I will illustrate how the dataflow transfers between your local machine and cloud cluster (dask-scheduler). There are serveral steps to follow:

- setp#1

  Open your local port 8080 in your browser you can find the airflow web GUI, the GUI will show you all dags in your local machine dags_folders.

- step#2

  Once you trigger a dag, your local airflow will access to the cloud dask-scheduler (recall that we assign the cloud-end dask-scheduler as the airflow executor/scheduler), and send the dag message to that dask-scheduler. According to the dag, cloud dask-scheduler will distribute all tasks inside this dag to all dask-workers.

  Technically, we can regard the procedure as a command line which dask-scheduler sends to dask-worker, for example:

  ```bash
  airflow run $dag_id $task_id $executeion_date --local -sd $PATH_TO_FIND_DAGS_ON_DASK_WORKER
  ```

  As you can see there, the dask-scheduler actually distributes one task to a specific dask-worker, and tells dask-worker where to find the dag difinition file via `-sd` parameter.

  - step#3

  Once all tasks in dag are done by dask-workers by order, the dask-scheduler will set the dag status to success.
