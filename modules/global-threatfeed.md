# Global Threadfeeds

**Goal:** Configure egress access control for outside threadfeed policy so workloads within cluster are not allow to external networkset

## Steps

1. Protect workloads with GlobalThreatfeed from known bad actors.

    Calicocloud offers [Global threat feed](https://docs.tigera.io/reference/resources/globalthreatfeed) resource to prevent known bad actors from accessing Kubernetes pods.

    ```bash
    kubectl get globalthreatfeeds
    ```

    >Output is 
    ```bash
    NAME                           CREATED AT
    alienvault.domainthreatfeeds   2021-09-28T15:01:33Z
    alienvault.ipthreatfeeds       2021-09-28T15:01:33Z
    ```

    You can get these domain/ip list from yaml file, the url would be:

    ```bash
    kubectl get globalthreatfeeds alienvault.domainthreatfeeds -ojson | jq -r '.spec.pull.http.url'

    kubectl get globalthreatfeeds alienvault.ipthreatfeeds -ojson | jq -r '.spec.pull.http.url'
    ```

    >Output is 
    ```bash
    https://installer.calicocloud.io/feeds/v1/domains

    https://installer.calicocloud.io/feeds/v1/ips
    ```


    ```bash
    # deploy feodo and snort threatfeeds
    kubectl apply -f demo/threatfeeds/feodo-tracker.yaml
    kubectl apply -f demo/threatfeeds/feodo-block-policy.yaml

    # Confirm and check the tracker threatfeed
    kubectl get globalthreatfeeds 

    ```

    ```bash
    NAME                           CREATED AT
    alienvault.domainthreatfeeds   2022-02-11T19:21:26Z
    alienvault.ipthreatfeeds       2022-02-11T19:21:26Z
    feodo-tracker                  2022-02-11T22:21:43Z 
    ```
    
2. Generate alerts by accessing the IP from `feodo-tracker` list. 

    ```bash
    # try to ping any of the IPs in from the feodo tracker list.
    FIP=$(kubectl get globalnetworkset threatfeed.feodo-tracker -ojson | jq -r '.spec.nets[0]' | sed -e 's/^"//' -e 's/"$//' -e 's/\/32//')
    kubectl -n dev exec -t netshoot -- sh -c "ping -c1 $FIP"
    ```

3. Generate alerts by accessing the IP from `alienvault.ipthreatfeeds` list. 

    ```bash
    # try to ping any of the IPs in from the ipthreatfeeds list.
    AIP=$(kubectl get globalnetworkset threatfeed.alienvault.ipthreatfeeds -ojson | jq -r '.spec.nets[0]' | sed -e 's/^"//' -e 's/"$//' -e 's/\/32//')
    kubectl -n dev exec -t netshoot -- sh -c "ping -c1 $AIP"
    ```


4. Add more threatfeeds into networkset and prevent your cluster from them.

    ```bash
    # deploy embargo and other threatfeeds
    kubectl apply -f demo/threatfeeds/embargo.networkset.yaml
    kubectl apply -f demo/threatfeeds/security.embargo-countries.yaml
    
    ```
    

5. . Confirm you are able to see the aler in alert list. 
   
     ![alert list](../img/alert-list.png)
        

---

[:arrow_right: Honeypods and Anomaly Detection](./intrusion-detection-protection.md)

[:leftwards_arrow_with_hook: Back to README.md](../README.md)