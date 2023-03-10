# Tigera Calico Cloud Workshop

## Join the Slack Channel

[Calico User Group Slack](https://slack.projectcalico.org/) is a great resource to ask any questions about Calico. If you are not a part of this Slack group yet, we highly recommend [joining it](https://slack.projectcalico.org/) to participate in discussions or ask questions. For example, you can ask questions specific to EKS and other managed Kubernetes services in the `#eks-aks-gke-iks` channel.

## Workshop objectives

The intent of this workshop is to guide users on connecting their K8s cluster to Calico Cloud and to test out its security and observability features.

## STEP 1 - Create a compatible k8s cluster 

  - [Calico Cloud system requirements](https://docs.calicocloud.io/get-started/connect/system-requirements)

  - [EKS: Use Cloud9 IDE to create a compatible EKS cluster](modules/creating-eks-cluster.md)
  - [AKS: Use Azure SDK to create a compatible AKS cluster](modules/creating-aks-cluster.md)
  - [GKE: Use gcloud SDK to create a compatible GKE cluster](modules/creating-gke-cluster.md)

  - [Kubeadm: Use kops to create a compatible self-managed cluster in GCP](modules/creating-kubeadm-cluster.md)
  - [RKE: Use Rancher server to create a compatible RKE cluster in GCP](modules/creating-rke-cluster.md)


## STEP 2 - Sign up for Calico Cloud  

  - Use this [link to register](https://www.calicocloud.io/) for a Calico Trial account

## STEP 3 - Connect your cluster to Calico Cloud

  - [Connect your cluster to Calico Cloud](modules/joining-calico-cloud.md)

## STEP 4 - Configure demo applications

  - [Configure your cluster and install demo applications](modules/configuring-demo-apps.md)

## STEP 5 - Test Calico Cloud features

Use cases:

### Network Policy for segmentation and access control

- [Segmentation and Workload Access Control](modules/app-service-control.md)
- [Egress Traffic Policy Based on FQDN](modules/dns-egress-controls.md)
- [Calico Cloud UI and Policy Management](modules/manager-ui.md)

### Observability

- [L7 logging](modules/enable-l7-visibility.md)
- [Service Graph](modules/manager-ui.md)
- [Kibana Dashboard](modules/kibana-dashboard.md)
- [Packet Capture](modules/dynamic-packet-capture.md) 

### Intrusion and Breach Detection

- [Global ThreatFeeds](modules/global-threatfeed.md)
- [Honeypods and Anomaly Detection](modules/intrusion-detection-protection.md)
- [Deep Packet Inspection](modules/deep-packet-inspection.md) 

### Compliance

- [Compliance reports](modules/compliance-reports.md) 

### Network Encryption with WireGuard

- [WireGuard Encryption](modules/encryption.md) 

## STEP 6 - Clean up your test environment

- [Clean Up](modules/clean-up.md)