# Module 5-3: East-West controls-Host protection

**Goal:** Secure EKS hosts ports with network policies.

Calico network policies not only can secure pod to pod communications but also can be applied to EKS hosts to protect host based services and ports. For more details refer to [Protect Kubernetes nodes](https://docs.tigera.io/security/kubernetes-nodes) documentaiton.

## Steps

1. Open a port of NodePort service for public access on EKS node.

    For the demo purpose we are going to expose the `default/frontend` service via the `NodePort` service type to open it for the public access.

    ```bash
    # expose the frontend service via the NodePort service type
    kubectl expose deployment frontend --type=NodePort --name=frontend-nodeport --overrides='{"apiVersion":"v1","spec":{"ports":[{"nodePort":30080,"port":80,"targetPort":8080}]}}'

    # open access to the port in AWS security group
    EKS_CLUSTER='jessie-workshop' # adjust the name if you used a different name for your EKS cluster
    AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
    # pick one EKS node and use it's ID to get securigy group
    SG_ID=$(aws ec2 describe-instances --region $AWS_REGION --filters "Name=tag:Name,Values=$EKS_CLUSTER*" "Name=instance-state-name,Values=running" --query 'Reservations[0].Instances[*].NetworkInterfaces[0].Groups[0].GroupId' --output text --output text)

    # get public IP of an EKS node
    PUB_IP=$(aws ec2 describe-instances --region $AWS_REGION --filters "Name=tag:Name,Values=$EKS_CLUSTER*" "Name=instance-state-name,Values=running" --query 'Reservations[0].Instances[0].PublicIpAddress' --output text --output text)
    
    # test connection to frontend 30080 port
    nc -zv $PUB_IP 30080

    # open SSH port in the security group for public access
    aws ec2 authorize-security-group-ingress --region $AWS_REGION --group-id $SG_ID --protocol tcp --port 30080 --cidr 0.0.0.0/0

    # test connection to frontend 30080 port again
    nc -zv $PUB_IP 30080
   
    ```

    >It can take a moment for the node port to become accessible.

    If the frontend service port was configured correctly, the `nc` command should show you that the port is open.

2. Enable `HostEndpoint` auto-creation for EKS cluster.

    When working with managed Kubernetes services, such as EKS, we recommend using `HostEndpoint` (HEP) auto-creation feature which allows you to automate the management of `HostEndpoint` resources for managed Kubernetes clusters whenever the cluster is scaled.

    
    ```bash
    # Before you enable HEP auto-creation feature, make sure there are no `HostEndpoint` resources manually defined for your cluster
    kubectl get hostendpoints
    ```

    ```bash
    # check whether auto-creation for HEPs is enabled. Default: Disabled
    kubectl get kubecontrollersconfiguration.p default -ojsonpath='{.status.runningConfig.controllers.node.hostEndpoint.autoCreate}'

    # enable HEP auto-creation
    kubectl patch kubecontrollersconfiguration.p default -p '{"spec": {"controllers": {"node": {"hostEndpoint": {"autoCreate": "Enabled"}}}}}'
    # verify that each node got a HostEndpoint resource created
    kubectl get hostendpoints
    ```

3. Implement a Calico policy to control access to the service of NodePort type.

    Deploy a policy that only allows access to the node port from the Cloud9 instance.

    ```bash
    # get the public IP from cloud9 shell
    echo $PUB_IP
    
    # from your local shell test connection to the node port, i.e. 30080, using netcat or telnet or other connectivity testing tool
    PUB_IP=XX.XX.XX.XX #export same public IP to your local shell and test 
    nc -zv $PUB_IP 30080
    #You should able to connect 30080 before applying hostendpoint policy

    # get public IP of Cloud9 instance in the Cloud9 shell
    CLOUD9_IP=$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4)
    
    # deploy HEP policy
    sed -e "s/\${CLOUD9_IP}/${CLOUD9_IP}\/32/g" demo/host-end-point/frontend-nodeport-access.yaml | kubectl apply -f -
    
    # test access from Cloud9 shell
    nc -zv $PUB_IP 30080 #expecting pass

    # test access from local shell again
    nc -zv $PUB_IP 30080 #expecting fail

    ```

    Once the policy is implemented, you should not be able to access the node port `30080` from your local shell, but you should be able to access it from the Cloud9 shell.

    >Note that in order to control access to the NodePort service, you need to enable `preDNAT` and `applyOnForward` policy settings.



[Next -> Module 6-1](../modules/egress-access-controls.md)

[Previous -> Module 5-2](../modules/microsegmentation.md)

[Menu](../README.md)