# OpenShift Virtualization

## Prerequisite

Test on v4.14 and will most likely work on 4.13 and 4.15.

1. OpenShift Cluster with Bare Metal Worker and ODF installed

2. OpenShift GitOps

3. OpenShift Pipelines

4. OpenShift Virtualization with HyperConverged App

5. Download Windows ISO,
either [Windows Server 2019](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2019)
or [Windows 10](https://www.microsoft.com/en-us/software-download/windows10ISO)

6. Namespace that is still hard coded.

    ```shell
    oc new-project chrisj
    ```

## HTTP Server

1. Setup permissions

    ```shell
    # argocd permissions
    oc adm policy add-cluster-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller
    oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n httpd-server
    oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n openshift-storage

    # scc
    oc create -f ./argocd/scc.yaml
    ```

2. Get ArgoCD password and log in

    ```shell
    oc extract secrets/openshift-gitops-cluster -n openshift-gitops --keys=admin.password --to -
    ```

3. Create App

    ```shell
    # wait 10 min
    oc create -f ./argocd/httpd-app.yaml
    ```

4. Upload ISO to httpd server

    ```shell
    # wait 10 min
    ISO_FILE=./win.iso  # assuming iso is named win.iso
    POD_NAME=$(oc get pods --selector=app=httpd-server -o jsonpath='{.items[0].metadata.name}' -n httpd-server)
    oc cp $ISO_FILE $POD_NAME:/opt/app-root/src -n httpd-server

    # ISO link becomes
    # http://httpd-server.httpd-server.svc.cluster.local:8080/win.iso
    ```

## RHEL9 GitOps

There is no pipeline for this yet, but we can demonstrate using ArgoCD to
create a RHEL VM.

```shell
oc create -f ./argocd/vms-app.yaml
# project: chrisj
# login username: redhat
# login password: 8etj2bea5fJ9
# also check secret/authorized-keys for ssh key for passwordless login
  # currently set to my public key
  # https://github.com/jkeam/kubevirt-gitops/blob/odf/vms/base/secret.yaml
```

## Pipeline

We will now setup a pipeline that can build a Windows Golden Image
in the form of a Data Volume that can be cloned for every new VM we want to
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

3. Create configmap for auto unattended config

    ```shell
    # make sure you're in the chrisj project
    oc apply -f https://raw.githubusercontent.com/jkeam/tekton-windows-pipeline/main/windows10autounattend.yaml
    ```

4. Create pipeline tasks

    ```shell
    VERSION=$(curl -s https://api.github.com/repos/kubevirt/kubevirt-tekton-tasks/releases | jq '.[] | select(.prerelease==false) | .tag_name' | sort -V | tail -n1 | tr -d '"')
    oc apply -f "https://github.com/kubevirt/kubevirt-tekton-tasks/releases/download/${VERSION}/kubevirt-tekton-tasks.yaml"
    ```

4. Create pipeline

    ```shell
    oc apply -f ./pipeline/windows10.yaml
    ```

5. Trigger the pipeline, set `winImageDownloadURL` to your iso name.
Also pipeline takes ~30min to run.

## Windows GitOps

1. Fork the [Windows Repo](https://github.com/jkeam/ocp-virt-windows-gitops)

2. Update the [PVC](https://github.com/jkeam/ocp-virt-windows-gitops/blob/main/windows/kustomization.yaml#L32)
in your fork to match your PVC name and check that into GitHub.

    ```shell
    # to get your PVC name
    oc get pvc -o name -n chrisj | grep windows-10-base | sed 's/persistentvolumeclaim\///g'
    ```

3. Create the ArgoCD Application

    ```shell
    # make sure to update the repoURL to point to your fork
    oc create -f ./argocd/windows-vms-app.yaml
    ```

4. Use OCP Web Console to connect to console 'changepassword'
and enable remote desktop connection

## Links

1. [Main reference](https://docs.google.com/document/d/1T_IxWWDcVLzaHbb46sPiMV8ieOiCg-9F0xkp67fpePo/edit)

2. [Upstream docs](https://kubevirt.io/2021/Automated-Windows-Installation-With-Tekton-Pipelines.html)

3. [Windows Customize Pipeline](https://github.com/kubevirt/kubevirt-tekton-tasks/tree/main/release/pipelines/windows-customize)

4. [Fancy Windows Auto Unattended XML that installs SQL Server and VS Code](https://github.com/kubevirt/kubevirt-tekton-tasks/blob/main/release/pipelines/windows-customize/configmaps/windows-customize-configmaps.yaml)

5. [Execute in VM](https://kubevirt.io/user-guide/virtual_machines/tekton_tasks/#execute-commands-in-virtual-machines)

6. [Windows Server 2019 ISO](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2019) - If you use this, you will need a new auto unattended.xml file as the one in this demo only works with Windows 10
