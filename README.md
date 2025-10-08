# OpenShift Virtualization

## Prerequisite

Tested on OpenShift 4.18.
The Experience OpenShift Virtualization Roadshow cluster is what this is tested on.
It will also have all of the prerequisite already installed for you.

1. OpenShift Cluster with Bare Metal Worker(s) with usable storage (RWX)

2. OpenShift GitOps

3. OpenShift Pipelines

4. OpenShift Virtualization with HyperConverged App

5. Download Windows ISO,
either [Windows Server 2019](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2019)
or [Windows 10](https://www.microsoft.com/en-us/software-download/windows10ISO)
or [Windows 11](https://www.microsoft.com/en-us/software-download/windows11)

## Setup

1. Fork this repo

2. Create a GitHub Personal Access Token (PAT) for that fork,
giving permissions to commit changes to the repo above

3. Update `./argocd/secret.yaml` with your PAT information

4. Create namespace where VMs and the HTTP Server will go

    ```shell
    oc new-project cluster-services
    oc new-project vms
    ```

## HTTP Server

1. Setup permissions

    ```shell
    # argocd permissions
    oc adm policy add-cluster-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller
    oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n cluster-services
    oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n vms
    oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n openshift-storage

    # scc
    oc create -f ./argocd/scc.yaml

    # pipeline permissions
    oc create -f ./argocd/secret.yaml
    oc edit sa/pipeline  # add `- name: git-secret` to secrets
    ```

2. Get ArgoCD password and log in

    ```shell
    oc extract secrets/openshift-gitops-cluster -n openshift-gitops --keys=admin.password --to -
    ```

3. Create App

    ```shell
    oc create -f ./argocd/httpd-app.yaml
    # wait ~10 min for completion
    ```

4. Upload ISOs to httpd server

    ```shell
    ISO_FILE=./Win10_22H2_English_x64v1.iso
    POD_NAME=$(oc get pods --selector=app=httpd-server -o jsonpath='{.items[0].metadata.name}' -n cluster-services)
    oc cp $ISO_FILE $POD_NAME:/opt/app-root/src -n cluster-services
    # wait ~10 min for completion
    # ISO link becomes
    #   http://httpd-server.cluster-services.svc.cluster.local:8080/Win10_22H2_English_x64v1.iso

    ISO_FILE=./Win11_25H2_English_x64.iso
    POD_NAME=$(oc get pods --selector=app=httpd-server -o jsonpath='{.items[0].metadata.name}' -n cluster-services)
    oc cp $ISO_FILE $POD_NAME:/opt/app-root/src -n cluster-services
    # wait ~10 min for completion
    # ISO link becomes
    #   http://httpd-server.cluster-services.svc.cluster.local:8080/Win11_25H2_English_x64.iso
    ```

## Fedora GitOps

Using ArgoCD to spin up a new Fedora VM.

```shell
oc create -f ./argocd/fedora.yaml
# project: vms
# login username: redhat
# login password: pxlh-pusf-qmte
```

## Pipeline

We will now setup a pipeline that can build a Windows Golden Image
in the form of a PVC that can be cloned for every new VM we want to
create.

Make sure NOT to run these pipelines at the same time.
The cleanup and termination of one pipelinerun will delete the resources
used by another concurrent pipelinerun as they name the configmaps the same.

### Windows 10

```shell
oc create -f ./pipeline/win10-pipelinerun.yaml
# wait 30min for completion
```

### Windows 11

```shell
oc create -f ./pipeline/win11-pipelinerun.yaml
# wait 30min for completion
```

## Windows GitOps

1. Create the ArgoCD Application for the VM you want

    ```shell
    # for windows 10 vm
    oc create -f ./argocd/win10.yaml

    # for windows 11 vm
    oc create -f ./argocd/win11.yaml
    ```

2. Use OCP Web Console to log into the Windows VM as `Administrator` and password `password` for win10 and `123456` for win11.

## Modify VM Config

You can modify the `sysprep.yaml` and/or `vm.yaml` files in the `vm/vm10` and `vm/vm11` dirs.

## Stamping out New Golden Images

1. Copy this [file](https://raw.githubusercontent.com/kubevirt/kubevirt-tekton-tasks/main/release/pipelines/windows-efi-installer/configmaps/windows-efi-installer-configmaps.yaml)

2. Add a new sysprep configmap into that file and keep note of the name of your new configmap, for example `customsysprep`

3. Find a place, like GitHub, to host that file, for example `https://gist.githubusercontent.com/YOURNAME/SOMEHASH/raw/SOMEHASH/windows-efi-installer-configmaps.yaml`

4. In `pipeline/win10-pipelinerun.yaml` if you want to update the Win10 golden image or `pipeline/win11-pipelinerun.yaml` if you want to update the Win11 golden image, update the following

    ```yaml
    - name: autounattendXMLConfigMapsURL
      value: https://gist.githubusercontent.com/YOURNAME/SOMEHASH/raw/SOMEHASH/windows-efi-installer-configmaps.yaml
    - name: autounattendConfigMapName
      value: customsysprep
    ```

5. Run the pipeline(s)

    ```shell
    # for a new win10 golden image
    oc create -f ./pipeline/win10-pipelinerun.yaml
    # wait 30min for completion

    # for a new win11 golden image
    oc create -f ./pipeline/win11-pipelinerun.yaml
    # wait 30min for completion
    ```

## Links

1. [Main reference](https://docs.google.com/document/d/1T_IxWWDcVLzaHbb46sPiMV8ieOiCg-9F0xkp67fpePo/edit)

2. [Upstream docs](https://kubevirt.io/2021/Automated-Windows-Installation-With-Tekton-Pipelines.html)

3. [Windows Customize Pipeline](https://github.com/kubevirt/kubevirt-tekton-tasks/tree/main/release/pipelines/windows-customize)

4. [Fancy Windows Auto Unattended XML that installs SQL Server and VS Code](https://github.com/kubevirt/kubevirt-tekton-tasks/blob/main/release/pipelines/windows-customize/configmaps/windows-customize-configmaps.yaml)

5. [Execute in VM](https://kubevirt.io/user-guide/virtual_machines/tekton_tasks/#execute-commands-in-virtual-machines)

6. [PipelineRuns](https://artifacthub.io/packages/tekton-pipeline/redhat-pipelines/windows-efi-installer#how-to-run)

7. [Example ConfigMaps](https://github.com/kubevirt/kubevirt-tekton-tasks/blob/main/release/pipelines/windows-customize/configmaps/windows-customize-configmaps.yaml)

8. [Blog Golden Image 2024](https://developers.redhat.com/articles/2024/09/09/create-windows-golden-image-openshift-virtualization#customize_the_installation_process)

9. [Blog Golden Image 2025](https://developers.redhat.com/articles/2025/08/12/windows-image-building-service-openshift-virtualization)
