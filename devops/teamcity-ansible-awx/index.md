[Introduction](#introduction) <br />

[Assignment](#assignment) <br />

[Topology](#topology) <br />

[Configuration](#configuration) <br />

  - [TeamCity](#teamcity) <br />
  - [Ansible](#ansible) <br />
  - [AWX](#awx) <br />

[Final Overview](#final-overview) <br />

## Introduction

This practice lab was designed with the goal of showcasing and teaching basic DevOps principles and technologies. 

This lab involves creating three virtual machines - two Windows machines and one Linux machine. Virtualization software used in this lab is Oracle VirtualBox and the recommended system requirements are at least 16 GB of RAM, quad core processor and 100 GB of hard disk space. Lower specs could be used if using Windows Server images for Windows machines. Better specs are always welcome. Configuration of VM's in VirtualBox won't be covered in this lab, but more detail can be found [here](https://www.virtualbox.org/manual/).

## Assignment

Set up TeamCity on the first Windows VM and Ansible/AWX on your Linux VM. Afterwards, create a configuration which will allow an automatic build of sample Windows service on TeamCity server from GitHub source, then deploy the build over Ansible server to the third (Windows/Client) virtual machine. Two administrator accounts need to be created on the client VM, and the deployed service needs to be executed in isolation under their respective accounts (simulating two destination servers). To recap:

1. Connect TeamCity to GitHub repository and perform build through TeamCity
2. Connect TeamCity to Ansible
3. Deploy build via Ansible to client VM as two isolated Windows services, which are run by separate Administrator accounts

Sources: <br />
https://www.jetbrains.com/teamcity/ <br />
https://github.com/MonoSoftware/sample-windows-service

## Topology

![Topology](/assets/images/devops/teamcity-ansible-awx/devops_topology.png)

VM01 - Teamcity <br />
VM02 - Ansible/AWX <br />
VM03 - Client <br />

## Configuration

### TeamCity

TeamCity is a Continuous Integration (CI) tool which, among other things, enables you to build software from source code, adequately test it and prepare it for deployment with a streamlined set of tools and processes. TeamCity Server is serving as a central control point for your TeamCity environment, while TeamCity Agents are used to perform the actual work of building/testing software. Since this is a simple lab, a single machine will serve as both Server and Agent - it is advised to deploy these roles on different machines in more complex environments.

Start by downloading and installing TeamCity Server on your first Windows VM. For this lab, all installation settings can remain on default selections. Once the installation completes, open a web browser and type `localhost:8111`. If you changed the default port during installation to something other than 8111, please use this port. Accept the license agreement and for database type select **Internal (HSQLDB)**. If you did not change the default selection for authentication during installation, you should be prompted to set up a password for the Super User local system account. After logging in, you will be greeted by the TeamCity WebUI.

We now need to create a new project (named **Sample Windows Service**) and link it to the relevant [GitHub repository](https://github.com/MonoSoftware/sample-windows-service).

After performing these steps, we can now proceed with build configuration. We will name the first build configuration within our project "**Build**" and will proceed to edit this configuration. You should notice that the VCS (Version Control System) details are already populated, as was specified on project creation. 

Next thing to note is the **Build Step** section - here we can see that TeamCity automatically detected a number of possible build steps by looking at the files in the linked GitHub repository. There should be four build steps automatically provided by TeamCity, two of which are **.bat** scripts used for installation/uninstallation of the relevant service. We will use only one provided build step - a .NET build step (using a .NET runner) which will use **msbuild** tools to perform the build. You should disable the other build steps:

![Build Steps](/assets/images/devops/teamcity-ansible-awx/build_steps.png)

Also, notice that the **Triggers** configuration is already populated by a VCS trigger, and the following parameter: `+:*`. This means that any addition to the linked GitHub repository will trigger a new build.

The remaining enabled .NET build step should already be correctly configured, but before we actually run the build, we still have a few more configuration items to update. First of these is **Arctifact Paths** within **General Settings** of the build configuration. If you don't specify this, the build will not produce artifacts in an expected way. TeamCity has special syntax for defining such configuration, which you can explore by looking into official TeamCity documentation. To complete this part of configuration, please enter the following:
```
+:**/* => target_directory.tar
-:*.gitattributes => target_directory.tar
-:*.gitignore => target_directory.tar
-:*.sln => target_directory.tar
```
In this case we opted to archive our build artifacts within a **.tar** archive named **target_directory**.

We are now ready to attempt running our build, which we can do by selecting **Run** in the upper right corner. After you select **Run**, an available agent will start working on the build and will most likely fail due to missing dependencies. This should be no cause for concern since these dependencies can be resolved easily by installing required software or modifying system environmental tables - in my case, I had to install the required .NET runtime, MSBuild Tools version and also update the PATH environmental variable to include the relevant **dotnet** executable path. After resolving all dependency issues, we should now be able to perform a clean build:

![Successful Build](/assets/images/devops/teamcity-ansible-awx/successful_build.png)

Having completed our build, we now need to push it to the Ansible server for further deployment. To achieve this, we will introduce a second build configuration in our TeamCity project, which we will name **Deploy**. Only one build step will be configured in this part and it will utilize the **SSH Upload** runner. For this to work, we also have to generate an SSH key pair, private and public. [Here](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement) is more info on SSH key pair generation. We will add the generated private key to TeamCity by visiting settings of our project, and adding the relevant private key. The public key will have to be added to Ansible server's **.ssh** directory, within the **authorized_keys** file.

Now we can proceed with configuring the **SSH Upload** build step within our **Deploy** build configuration. Here is a short overview of the configuration:

![SSH Upload](/assets/images/devops/teamcity-ansible-awx/ssh_upload.png)

**Target** represents the destination server (in this case Ansible server) and the directory where we want the artifacts uploaded. You can leave protocol on **SCP**, then enter select the SSH key you added earlier from the dropdown. Credentials for relevant account on Ansible server should be entered. Please note that in this example the root account was used - this is a bad security practice and you would usually create and use a dedicated account for this type of access. We opted for the root account in this lab environment. Lastly define **Path to sources** which should point to the location of our artifact(s). In this case I entered **target_directory.tar**.

Last but not least, we also have to define a dependency for our **Deploy** build configuration step - after all, we want the SSH Upload to be performed only after a successful build. Go to **Dependencies** and here add a new artifact dependecy. **Depend on** should be populated by the build configuration named "**Build**" and the **Artifact rules** should reflect the source and destination of the relevant artifact - in our case this parameter is populated like this: `target_directory.tar => target_directory.tar`.

Now let's try running our Build and Deploy:

![Successful Push](/assets/images/devops/teamcity-ansible-awx/successful_push.gif)

Finally, let's check our destination server (Ansible):

![SSH Upload Confirm](/assets/images/devops/teamcity-ansible-awx/ssh_upload_confirm.png)

Great news, the artifacts archive was successfully transfered to the Ansible server and is ready to be further processed/deployed!

Last piece of configuration which we need to perform on the TeamCity server is to configure an SSH Exec build step in our **Deploy** build configuration, which will execute the required bash script and/or Ansible commands, which will enable further deployment to the Client machine.

### Ansible

Now that we have the build artifacts, we need to create an Ansible Playbook which will perform the transfer of the archive to the Client machine, unarchiving of the file and running relevant Windows command line commands which will sanitize the transfer folder and deploy the services under two separate Windows accounts.

We can start our configuration by installing Ansible - this can be done by using the `pip` Python package installer, or your Linux distribution's native package manager. Recommended way is using `pip`, since this method will provide the latest packages.

After installing Ansible, it's time to configure our Inventory file - here we will define all client machines which Ansible will administer. Make sure to create the file within the same directory where your Playbooks and configuration files will be stored. The syntax of the file is simple, which is a list of IP addresses and/or hostnames of the machines you wish to be managed by Ansible, alongside any relevant global config. Example:

```
[win]
192.168.178.45

[win:vars]
ansible_user=Kreso
ansible_password=Password123
ansible_connection=winrm
ansible_winrm_server_cert_validation=ignore
```

Next it's time to create an Ansible configuration file. Here we will define global Ansible settings which will enable us to shorten our Ansible commands and make overall quality of life improvements. Name the file **Ansible.cfg** and populate it as needed. Example:

```
[defaults]
inventory = inventory
private_key_file = ~/.ssh/id_rsa
remote_user = root
```
This simple file outlines the structure of the file and some basic settings we can configure. In this case we defined which file will be used as an Inventory file (file we created earlier and named "**Inventory**"),  specified the location of a relevant SSH private key file and configured a default user when connecting to remote hosts (in this case **root**).

Finally, we have to configure a Playbook, which is a YAML file containing the necessary tasks we want to be performed. An example YAML file:

```
---

- name: Run Playbook
  hosts: win
  ignore_errors: true

  tasks:

  - name: Stop service1
    win_service:
      name: Mono-Sample-Windows-Service1
      state: stopped

  - name: Stop service2
    win_service:
      name: Mono-Sample-Windows-Service2
      state: stopped

  - name: Remove SampleService folder
    ansible.windows.win_file:
     path: C:\SampleService\
     state: absent

  - name: Create SampleService folder
    ansible.windows.win_file:
     path: C:\SampleService
     state: directory

  - name: Copy a single file to Windows host
    ansible.windows.win_copy:
      src: /root/Ansible/dotnetapp/target_directory.zip
      dest: C:\SampleService\

  - name: Unzip file
    community.windows.win_unzip:
      src: C:\SampleService\target_directory.zip
      dest: C:\SampleService

  - name: Rename file
    win_command: 'cmd.exe /c rename "C:\SampleService\WindowsService" WindowsService1'

  - name: Copy a single file
    ansible.windows.win_copy:
      remote_src: true
      src: C:\SampleService\WindowsService1
      dest: C:\SampleService\WindowsService2

  - name: Remove archive
    ansible.windows.win_file:
     path: C:\SampleService\target_directory.zip
     state: absent

  - name: Remove service1
    win_service:
      name: Mono-Sample-Windows-Service1
      state: absent

  - name: Create Service1
    ansible.windows.win_service:
     name: Mono-Sample-Windows-Service1
     path: C:\SampleService\WindowsService1\bin\Debug\SampleWindowsService.exe
     display_name: Mono-Sample-Windows-Service1

  - name: Set user for Service1
    ansible.windows.win_service:
      name: Mono-Sample-Windows-Service1
      state: restarted
      username: .\sampleservice1
      password: Password123

  - name: Remove service2
    win_service:
      name: Mono-Sample-Windows-Service2
      state: absent

  - name: Create Service2
    ansible.windows.win_service:
     name: Mono-Sample-Windows-Service2
     path: C:\SampleService\WindowsService1\bin\Debug\SampleWindowsService.exe
     display_name: Mono-Sample-Windows-Service2

  - name: Set user for Service2
    ansible.windows.win_service:
      name: Mono-Sample-Windows-Service2
      state: restarted
      username: .\sampleservice2
      password: Password123
```

We can now apply the TeamCity SSH Exec deploy step which will run the Ansible command which will initiate the playbook. For example:

`ansible-playbook -i inventory test.yaml`

### AWX

AWX provides a web-based user interface, REST API, and task engine built on top of Ansible. It is one of the upstream projects for Red Hat Ansible Automation Platform. Integrating AWX in our CI/CD process will enable us to achieve better visibility of project health, help us to manage secrets more efficiently and provide more streamlined control over inventory/job configuration.

Latest versions of AWX are distributed in a way which requires configuration of several containers/pods, and Kubernetes-like tools like **k3s** will help us to complete this installation. We will use Ubuntu 22.04 LTS for deploying AWX.

#### Installation

**Step 1: Update Ubuntu system**

```
sudo apt update && sudo apt -y upgrade
[ -f /var/run/reboot-required ] && sudo reboot -f
```

**Step 2: Install Single Node k3s Kubernetes**

We will deploy a single node kubernetes using k3s lightweight tool. K3s is a certified Kubernetes distribution designed for production workloads in unattended, resource-constrained environments. The good thing with k3s is that you can add more Worker nodes at later stage if need arises.

K3s provides an installation script that is a convenient way to install it as a service on systemd or openrc based systems
Let’s run the following command to install K3s on our Ubuntu system:

```
curl -sfL https://get.k3s.io | sudo bash -
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```
Validate K3s installation:

The next step is to validate our installation of K3s using kubectl command which was installed and configured by installer script.

```
$ kubectl get nodes

NAME    STATUS   ROLES                  AGE   VERSION
jammy   Ready    control-plane,master   58s   v1.25.6+k3s1
```
You can also confirm Kubernetes version deployed using the following command:

```
$ kubectl version --short
Client Version: v1.25.6+k3s1
Kustomize Version: v4.5.7
Server Version: v1.25.6+k3s1
```

The K3s service will be configured to automatically restart after node reboots or if the process crashes or is killed.

**Step 3: Deploy AWX Operator on Kubernetes**

This Kubernetes Operator has to be deployed in your Kubernetes cluster, which in our case is powered by K3s. The operator we’ll deploy can manage one or more AWX instances in any namespace.
Install git and make tools:

```
sudo apt update
sudo apt install git build-essential -y
```

Clone operator deployment code:

```
$ git clone https://github.com/ansible/awx-operator.git
Cloning into 'awx-operator'...
remote: Enumerating objects: 6306, done.
remote: Counting objects: 100% (315/315), done.
remote: Compressing objects: 100% (189/189), done.
remote: Total 6306 (delta 160), reused 236 (delta 115), pack-reused 5991
Receiving objects: 100% (6306/6306), 1.54 MiB | 19.74 MiB/s, done.
Resolving deltas: 100% (3598/3598), done.
```

Create namespace where operator will be deployed. I’ll name mine awx:
```
export NAMESPACE=awx
kubectl create ns ${NAMESPACE}
```

Set current context to value set in NAMESPACE variable:

```
# kubectl config set-context --current --namespace=$NAMESPACE 
Context "default" modified.
```
Switch to awx-operator directory:
```
cd awx-operator/
```
Save the latest version from AWX Operator releases as RELEASE_TAG variable then checkout to the branch using git.
```
sudo apt install curl jq -y
RELEASE_TAG=`curl -s https://api.github.com/repos/ansible/awx-operator/releases/latest | grep tag_name | cut -d '"' -f 4`
echo $RELEASE_TAG
git checkout $RELEASE_TAG
```
Deploy AWX Operator into your cluster:
```
export NAMESPACE=awx
make deploy
```

Wait a few minutes and awx-operator should be running:
```
# kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
awx-operator-controller-manager-68d787cfbd-z75n4   2/2     Running   0          40s
```

How To Uninstall AWX Operator (Don’t run this unless you’re sure it uninstalls!“

You can always remove the operator and all associated CRDs by running the command below:

```
# export NAMESPACE=awx
# make undeploy
/root/awx-operator/bin/kustomize build config/default | kubectl delete -f -
namespace "awx" deleted
customresourcedefinition.apiextensions.k8s.io "awxbackups.awx.ansible.com" deleted
customresourcedefinition.apiextensions.k8s.io "awxrestores.awx.ansible.com" deleted
customresourcedefinition.apiextensions.k8s.io "awxs.awx.ansible.com" deleted
serviceaccount "awx-operator-controller-manager" deleted
role.rbac.authorization.k8s.io "awx-operator-leader-election-role" deleted
role.rbac.authorization.k8s.io "awx-operator-manager-role" deleted
clusterrole.rbac.authorization.k8s.io "awx-operator-metrics-reader" deleted
clusterrole.rbac.authorization.k8s.io "awx-operator-proxy-role" deleted
rolebinding.rbac.authorization.k8s.io "awx-operator-leader-election-rolebinding" deleted
rolebinding.rbac.authorization.k8s.io "awx-operator-manager-rolebinding" deleted
clusterrolebinding.rbac.authorization.k8s.io "awx-operator-proxy-rolebinding" deleted
configmap "awx-operator-manager-config" deleted
service "awx-operator-controller-manager-metrics-service" deleted
deployment.apps "awx-operator-controller-manager" deleted
```

**Step 4: Install Ansible AWX on Ubuntu using Operator**

Create Static data PVC – Ref AWX data persistence:
```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: static-data-pvc
  namespace: awx
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 5Gi
EOF
```
PVC won’t be bound until the pod that uses it is created.
```
# kubectl get pvc -n awx
NAME                     STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
static-data-pvc          Pending                                      local-path     43s
```
Let’s now create AWX deployment file with basic information about what is installed:

**vim awx-deploy.yml**

Paste below contents into the file:
```
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  service_type: nodeport
  projects_persistence: true
  projects_storage_access_mode: ReadWriteOnce
  web_extra_volume_mounts: |
    - name: static-data
      mountPath: /var/lib/projects
  extra_volumes: |
    - name: static-data
      persistentVolumeClaim:
        claimName: static-data-pvc
```
We have defined resource name as awx and service type as nodeport to enable us access AWX from the Node IP address and given port. We also added extra PV mount on the web server pod.

Apply configuration manifest file:
```
$ kubectl apply -f awx-deploy.yml
awx.awx.ansible.com/awx created
```
Wait a few minutes then check AWX instance deployed:
```
$ watch kubectl get pods -l "app.kubernetes.io/managed-by=awx-operator"
NAME                   READY   STATUS    RESTARTS   AGE
awx-postgres-0         1/1     Running   0          75s
awx-7c5d846c88-mjlvm   4/4     Running   0          64s
```
You can track the installation process at the operator pod logs:
```
kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager
```

Data Persistence

The database data will be persistent as they are stored in a persistent volume:
```
$ kubectl get pvc
NAME                      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
postgres-awx-postgres-0   Bound    pvc-c0149545-8631-4aa1-a03f-2134a7f42aa6   8Gi        RWO            local-path     84s
static-data-pvc           Bound    pvc-6b6005de-0888-4634-b0a2-d5cc92eb85cc   1Gi        RWO            local-path     2m10s
awx-projects-claim        Bound    pvc-91e751e9-0e8e-40c8-9953-f8d9db5f612b   8Gi        RWO            local-path     77s
```
Volumes are created using local-path-provisioner and host path
```
$ ls /var/lib/rancher/k3s/storage/
pvc-edb29795-7dae-4a00-805f-2d989694fe3d_default_postgres-awx-postgres-0
```
Checking AWX Container’s logs
The awx-xxx-yyy pod will have four containers, namely:

redis
awx-web
awx-task
awx-ee
As can be seen from below command output:
```
# kubectl -n awx  logs deploy/awx
error: a container name must be specified for pod awx-75698588d6-r7bxl, choose one of: [redis awx-web awx-task awx-ee]
```
You’ll need to provide container name after the pod:

`kubectl -n awx  logs deploy/awx -c redis`
`kubectl -n awx  logs deploy/awx -c awx-web`
`kubectl -n awx  logs deploy/awx -c awx-task`
`kubectl -n awx  logs deploy/awx -c awx-ee`

Access AWX Container’s Shell

Here is how to access each container’s shell:

`kubectl exec -ti deploy/awx  -c  awx-task -- /bin/bash`
`kubectl exec -ti deploy/awx  -c  awx-web -- /bin/bash`
`kubectl exec -ti deploy/awx  -c  awx-ee -- /bin/bash`
`kubectl exec -ti deploy/awx  -c  redis -- /bin/bash`


**Step 5: Access Ansible AWX Dashboard**

List all available services and check awx-service Nodeport
```
$ kubectl get svc -l "app.kubernetes.io/managed-by=awx-operator"
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
awx-postgres-13   ClusterIP   None            <none>        5432/TCP       153m
awx-service       NodePort    10.43.179.217   <none>        80:30080/TCP   152m
```
We can see the service type is set to NodePort. To access our service from outside Kubernetes cluster using k3s node IP address and give node port

http://hostip_or_hostname:30080
You can edit the Node Port and set to figure of your preference, but it has to be in the range of 30000-32768

On accessing AWX web portal you are presented with the welcome dashboard similar to one below.

![AWX Welcome Screen](/assets/images/devops/teamcity-ansible-awx/awx_welcomescreen.png)

Obtain admin user password by decoding the secret with the password value:

`kubectl get secret awx-admin-password -o jsonpath="{.data.password}" | base64 --decode`

Better output format:

`kubectl get secret awx-admin-password -o go-template='{{range $k,$v := .data}}{{printf "%s: " $k}}{{if not $v}}{{$v}}{{else}}{{$v | base64decode}}{{end}}{{"\n"}}{{end}}'`

#### Deployment

Having installed AWX, we can now configure our project and job template.

First log in to the AWX web UI if you did not already do so and open **Inventories**. Here, we will create a new inventory add two hosts - TeamCity and Client:

![AWX Hosts](/assets/images/devops/teamcity-ansible-awx/awx_hosts.png)

Additional variables can be set at inventory level or single host level - in our case we configured login method and credential for our Windows client:

![AWX Host Details](/assets/images/devops/teamcity-ansible-awx/awx_hostdetails.png)

Next, we'll set up our credentials, so select **Credentials**. Here we will add two credentials - one for our AWX host machine and another for GitHub.

AWX host machine:

![AWX Host Credential](/assets/images/devops/teamcity-ansible-awx/credential_awxhost.png)

GitHub:

![GitHub Credential](/assets/images/devops/teamcity-ansible-awx/credential_github.png)

*Note: for the GitHub credential, make sure to select type "Source Control", then generate a new Personal Access Token (PAT) within GitHub Settings -> Developer Settings, and use this as the password.*

Next configure a new project, similar to this example:

![GitHub Credential](/assets/images/devops/teamcity-ansible-awx/credential_github.png)

Make sure to **Sync** the project, in order to check if a connection with your GitHub repository was successful.

Now it's time to configure a new job template, which we will **Launch** to start our playbook. Here's a reference configuration:

![AWX Job Template](/assets/images/devops/teamcity-ansible-awx/awx_jobtemplate.png)

And here's the YAML file which will be our playbook:

```
---

- name: Pre-run

  hosts: 192.168.178.47
  ignore_errors: true
  gather_facts: no
  
  tasks:
  - name: Fetch file from host to pod
  
    ansible.builtin.fetch:
        src: /home/krebor/Ansible/dotnetapp/target_directory.zip
        dest: /tmp/target_directory.zip
        flat: yes
        
- name: Run playbook
  
  hosts: 192.168.178.45
  ignore_errors: true
  gather_facts: no

  tasks:
        
  - name: Stop service1
    win_service:
      name: Mono-Sample-Windows-Service1
      state: stopped

  - name: Stop service2
    win_service:
      name: Mono-Sample-Windows-Service2
      state: stopped

  - name: Remove SampleService folder
    ansible.windows.win_file:
     path: C:\SampleService\
     state: absent

  - name: Create SampleService folder
    ansible.windows.win_file:
     path: C:\SampleService
     state: directory

  - name: Copy a single file to Windows host
    ansible.windows.win_copy:
      src: /tmp/target_directory.zip
      dest: C:\SampleService\

  - name: Unzip file
    community.windows.win_unzip:
      src: C:\SampleService\target_directory.zip
      dest: C:\SampleService

  - name: Rename file
    win_command: 'cmd.exe /c rename "C:\SampleService\WindowsService" WindowsService1'

  - name: Copy a single file
    ansible.windows.win_copy:
      remote_src: true
      src: C:\SampleService\WindowsService1
      dest: C:\SampleService\WindowsService2

  - name: Remove archive
    ansible.windows.win_file:
     path: C:\SampleService\target_directory.zip
     state: absent

  - name: Remove service1
    win_service:
      name: Mono-Sample-Windows-Service1
      state: absent

  - name: Create Service1
    ansible.windows.win_service:
     name: Mono-Sample-Windows-Service1
     path: C:\SampleService\WindowsService1\bin\Debug\SampleWindowsService.exe
     display_name: Mono-Sample-Windows-Service1

  - name: Set user for Service1
    ansible.windows.win_service:
      name: Mono-Sample-Windows-Service1
      state: restarted
      username: .\sampleservice1
      password: Password123

  - name: Remove service2
    win_service:
      name: Mono-Sample-Windows-Service2
      state: absent

  - name: Create Service2
    ansible.windows.win_service:
     name: Mono-Sample-Windows-Service2
     path: C:\SampleService\WindowsService1\bin\Debug\SampleWindowsService.exe
     display_name: Mono-Sample-Windows-Service2

  - name: Set user for Service2
    ansible.windows.win_service:
      name: Mono-Sample-Windows-Service2
      state: restarted
      username: .\sampleservice2
      password: Password123
```

We now have some additional configuration we need to take care of before we are able to run the job template. First will be exposing the local filesystem directory of host machine (which is hosting AWX) to the awx-ee pod, so that files which are pushed from TeamCity can become available to the pod. We can do so by going to **Settings** within AWX web UI and under job settings populate **Paths to expose to isolated jobs**. Here's an example config:

```
[
  "/etc/pki/ca-trust:/etc/pki/ca-trust:O",
  "/home/krebor/Ansible/dotnetapp/:/tmp:rw",
  "/usr/share/pki:/usr/share/pki:O"
]
```

We also have to make sure that relevant additional Ansible modules can be installed/provisioned within the pod and we can do so by creating a requirements.yml file under collections within our linked GitHub repository. Sample config:

```
collections:
- name: ansible.windows
- name: community.windows
```

We can now run the job template successfully:

![AWX Successful Job](/assets/images/devops/teamcity-ansible-awx/awx_successfuljob.png)

Lastly, we need to reconfigure TeamCity Deploy steps (SSH Upload and SSH Exec) to reflect our new setup. Update the hostname to the hostname or IP address of AWX host machine, update the credentials and for SSH Exec input a command which will run a tower-cli (which you may also need to install beforehand) and subsequently the AWX job template. Sample command:

`sudo tower-cli job launch -J "sample-job" --monitor -f human`

And that should be it, you should now be able to successfully triger a build from TeamCity, which will push the build artifacts to the Ansible host machine, which will in turn use AWX to deploy the artifacts to the Windows client, as specified by the playbook.

References:

https://computingforgeeks.com/how-to-install-ansible-awx-on-ubuntu-linux/ - awx install process using k3s

https://www.ansible.com/blog/connecting-to-a-windows-host - setting up WinRM for Ansible

https://www.reddit.com/r/awx/comments/ut3usv/importing_custom_modules_into_ansible_awx/ - importing custom Ansible modules in awx

https://www.reddit.com/r/ansible/comments/zaiahe/using_fetch_module_to_copy_remote_files_to_local/ - issues with fetch

Demo Videos:

https://youtu.be/Vx96Wwn004Y

https://youtu.be/GAYFD_r4Stg


## Final Overview

What did we achieve in creating this lab? In short, we enabled an automatic build and deployment of newly commited source code, and even if we did not directly implement any tests, we saw how this could also be implemented as one of the build configurations within TeamCity. We leveraged our knowledge of virtualization and explored new concepts like TeamCity Server configuration and build/deploy process, Ansible/AWX configuration automation and containerization. Hopefully this was a valuable experience to spark further study and self-development.