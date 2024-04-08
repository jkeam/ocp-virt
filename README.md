# OpenShift Virtualization

## Prerequisite

Test on v4.14 and will most likely work on 4.13 and 4.15.

1. OpenShift Cluster with Bare Metal Worker and Default Storage Class
with Volume Binding Mode of `Immediate`

2. OpenShift GitOps

3. OpenShift Pipelines

4. OpenShift Virtualization with HyperConverged App

5. Aws Load Balancer Operator with

    ```shell
    oc apply -f ./awsloadbalancercontroller.yaml
    ```

6. Download Windows ISO,
either [Windows Server 2019](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2019)
or [Windows 10](https://www.microsoft.com/en-us/software-download/windows10ISO)

7. Namespace that is still hard coded.

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
    ISO_FILE=/Users/jkeam/dev/images/Win10_22H2_English_x64v1.iso
    POD_NAME=$(oc get pods --selector=app=httpd-server -o jsonpath='{.items[0].metadata.name}' -n httpd-server)
    oc cp $ISO_FILE $POD_NAME:/opt/app-root/src -n httpd-server

    # ISO link becomes
    # http://httpd-server.httpd-server.svc.cluster.local:8080/Win10_22H2_English_x64v1.iso
    ```

## GitOps

1. Create RHEL9 VM

    ```shell
    oc create -f ./argocd/vms-app.yaml
    ```

## Pipeline

1. Configure GitOps rbac

    ```shell
    git clone git@github.com:OOsemka/gitops-acm1.git
    cd ./gitops-acm1/components/operators/openshift-gitops/instance/overlays/default
    oc create -k .
    ```

2. Create project all this will live in

    ```shell
    oc new-project chrisj
    ```

3. Create configmap for auto unattended config

    ```shell
    oc apply -f https://raw.githubusercontent.com/OOsemka/tekton-windows-pipeline/main/windows10autounattend.yaml
    ```

4. Create pipeline tasks

    ```shell
    VERSION=$(curl -s https://api.github.com/repos/kubevirt/kubevirt-tekton-tasks/releases | jq '.[] | select(.prerelease==false) | .tag_name' | sort -V | tail -n1 | tr -d '"')
    kubectl apply -f "https://github.com/kubevirt/kubevirt-tekton-tasks/releases/download/${VERSION}/kubevirt-tekton-tasks.yaml"
    ```

5. Create pipeline

    ```shell
    # wait 10 min
    oc apply -f https://raw.githubusercontent.com/OOsemka/tekton-windows-pipeline/main/windows10pipeline-fixed.yaml
    ```

6. Trigger the pipeline

7. Create Windows VM

    ```shell
    oc create -k ./windows
    ```

8. Start VM

    ```shell
    # use the OCP Web Console or the cli below
    virtctl start windows-10-vm
    ```

9. Use OCP Web Console to connect to console 'changepassword'
and enable remote desktop connection

## Links

1. [Main reference](https://docs.google.com/document/d/1T_IxWWDcVLzaHbb46sPiMV8ieOiCg-9F0xkp67fpePo/edit)

2. [Upstream docs](https://kubevirt.io/2021/Automated-Windows-Installation-With-Tekton-Pipelines.html)
