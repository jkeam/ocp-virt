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

1. Fork the [Windows Repo](https://github.com/jkeam/ocp-virt-windows-gitops)

2. Create a GitHub Personal Access Token (PAT) for that fork,
giving permissions to commit changes to the repo above

3. Update `./argocd/secret.yaml` with your PAT information

4. Create namespace where VMs will go

    ```shell
    oc new-project vms
    oc new-project cluster-services
    ```

## HTTP Server

1. Setup permissions

    ```shell
    # argocd permissions
    oc adm policy add-cluster-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller
    oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n cluster-services
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

4. Upload ISO to httpd server

    ```shell
    ISO_FILE=./win10.iso  # assuming iso is named win10.iso
    POD_NAME=$(oc get pods --selector=app=httpd-server -o jsonpath='{.items[0].metadata.name}' -n cluster-services)
    oc cp $ISO_FILE $POD_NAME:/opt/app-root/src -n cluster-services
    # wait ~10 min for completion

    # ISO link becomes
    # http://httpd-server.cluster-services.svc.cluster.local:8080/win10.iso
    ```

## RHEL9 GitOps

There is no pipeline for this yet, but we can demonstrate using ArgoCD to
create a RHEL VM.

```shell
oc create -f ./argocd/vms-app.yaml
# project: vms
# login username: redhat
# login password: 8etj2bea5fJ9
# also check secret/authorized-keys for ssh key for passwordless login
  # currently set to my public key
  # https://github.com/jkeam/kubevirt-gitops/blob/odf/vms/base/secret.yaml
```

## Pipeline

We will now setup a pipeline that can build a Windows Golden Image
in the form of a PVC that can be cloned for every new VM we want to
create.

1. Configure GitOps RBAC

    ```shell
    git clone git@github.com:OOsemka/gitops-acm1.git
    cd ./gitops-acm1/components/operators/openshift-gitops/instance/overlays/default
    oc create -k .  # after done, cd back to the ocp-virt repo home
    ```

2. Create configmap for auto unattended config

    ```shell
    oc create -f ./pipeline/windows10autounattend.yaml
    ```

3. Create pipeline tasks

    ```shell
    VERSION=$(curl -s https://api.github.com/repos/kubevirt/kubevirt-tekton-tasks/releases | jq '.[] | select(.prerelease==false) | .tag_name' | sort -V | tail -n1 | tr -d '"')
    oc apply -f "https://github.com/kubevirt/kubevirt-tekton-tasks/releases/download/${VERSION}/kubevirt-tekton-tasks.yaml"

    # custom task
    oc create -f ./pipeline/git-update-deployment-task.yaml
    ```

4. Create pipeline

    ```shell
    oc apply -f ./pipeline/windows10-pipeline.yaml
    ```

5. Trigger the pipeline, setting `WIN_IMAGE_DL_URL` and `GIT_REPOSITORY` params first

    ```shell
    # WIN_IMAGE_DL_URL: http://httpd-server.cluster-services.svc.cluster.local:8080/win10.iso
    # GIT_REPOSITORY: https://github.com/YOUR_NAME/ocp-virt-windows-gitops.git
    oc create -f ./pipeline/windows10-pipelinerun.yaml
    # wait ~30min for completion
    ```

6. Check to see a PVC named `windows-10-base-xxxx` was created

7. Check the repo at
`https://github.com/YOUR_NAME/ocp-virt-windows-gitops/blob/main/windows/patch.yaml`
and see the PVC on line 21 matches

## Windows GitOps

1. Create the ArgoCD Application, setting the `repoURL` first

    ```shell
    # repoURL: https://github.com/YOUR_NAME/ocp-virt-windows-gitops
    oc create -f ./argocd/windows-vms-app.yaml
    ```

2. Use OCP Web Console to login as `Administrator` and password `changepassword`

## Stamping out New Windows Images

1. Make some changes to the Windows Golden Image in the `configmap/windows-10-autounattend`

2. Trigger the pipeline, making sure `WIN_IMAGE_DL_URL` and `GIT_REPOSITORY` params are set

    ```shell
    # WIN_IMAGE_DL_URL: http://cluster-services.cluster-services.svc.cluster.local:8080/win10.iso
    # GIT_REPOSITORY: https://github.com/YOUR_NAME/ocp-virt-windows-gitops.git
    oc create -f ./pipeline/windows10-pipelinerun.yaml
    # wait ~30min for completion
    ```

3. Stop and destroy currently running VM `windows-10-vm`, wait a few min for completion

4. Manually `Sync` in ArgoCD to create new VM off new golden image

## Links

1. [Main reference](https://docs.google.com/document/d/1T_IxWWDcVLzaHbb46sPiMV8ieOiCg-9F0xkp67fpePo/edit)

2. [Upstream docs](https://kubevirt.io/2021/Automated-Windows-Installation-With-Tekton-Pipelines.html)

3. [Windows Customize Pipeline](https://github.com/kubevirt/kubevirt-tekton-tasks/tree/main/release/pipelines/windows-customize)

4. [Fancy Windows Auto Unattended XML that installs SQL Server and VS Code](https://github.com/kubevirt/kubevirt-tekton-tasks/blob/main/release/pipelines/windows-customize/configmaps/windows-customize-configmaps.yaml)

5. [Execute in VM](https://kubevirt.io/user-guide/virtual_machines/tekton_tasks/#execute-commands-in-virtual-machines)

6. [Windows Server 2019 ISO](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2019) - If you use this, you will need a new auto unattended.xml file as the one in this demo only works with Windows 10
