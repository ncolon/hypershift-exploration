# HyperShift Exploration (AWS Tech Preview)

## Happy Path

1. Deploy RHACM and create a MultiClusterHub. Make sure `multicluster-engine` is set to `enabled: true`

    ```yaml
    apiVersion: v1
    items:
    - apiVersion: operator.open-cluster-management.io/v1
      kind: MultiClusterHub
      metadata:
        name: multiclusterhub
        namespace: open-cluster-management
      spec:
        availabilityConfig: High
        enableClusterBackup: false
        overrides:
          components:
          - enabled: true
            name: multiclusterhub-repo
          - enabled: true
            name: management-ingress
          - enabled: true
            name: console
          - enabled: true
            name: insights
          - enabled: true
            name: grc
          - enabled: true
            name: cluster-lifecycle
          - enabled: true
            name: volsync
          - enabled: true
            name: multicluster-engine
          - enabled: true
            name: search
          - enabled: false
            name: cluster-backup
        separateCertificateManagement: true
    ```

2. Patch the default MultiClusterEngine to enable hypershift

    ```bash
    $ oc patch mce multiclusterengine --type=merge \
      -p  '{"spec":{"overrides":{"components":[{"name":"hypershift-preview" "enabled": true}]}}}'
    ```

3. If you’re deploying HyperShift on the same cluster where RHACM is enabled, make sure local-cluster is managed by RHACM

    ```yaml
    apiVersion: cluster.open-cluster-management.io/v1
    kind: ManagedCluster
    metadata:
      labels:
        local-cluster: "true"
      name: local-cluster
    spec:
      hubAcceptsClient: true
      leaseDurationSeconds: 60
    ```

    Q1: And if deploying on MCE standalone w/o RHACM?

4. Create an S3 bucket in the same region as your cluster

    ```bash
    export AWS_ACCESS_KEY_ID="youraccesskey"
    export AWS_SECRET_ACCESS_KEY="yoursecretaccesskey"
    BUCKET_NAME=your-bucket-name
    REGION=us-east-2
    aws s3api create-bucket --acl public-read --bucket $BUCKET_NAME \
      --create-bucket-configuration LocationConstraint=$REGION \
      --region $REGION
    ```

    Q2: Does the bucket need to be public? Why?

5. Create a secret with credentials to the hypershift bucket above, if you’re using another managed cluster other than `local-cluster` to manage HyperShift deployments replace `-n local-cluster` with your managed cluster namespace.

    ```yaml
    BUCKET_NAME=your-bucket-name
    REGION=us-east-2
    oc create secret generic hypershift-operator-oidc-provider-s3-credentials \
      --from-file=credentials=$HOME/.aws/credentials \
      --from-literal=bucket=$BUCKET_NAME \
      --from-literal=region=$REGION -n local-cluster
    ```

6. Create a ManagedClusterAddon to enable HyperShift, if you’re using another managed cluster other than local-cluster to manage HyperShift deployments replace “-n local-cluster” with your managed cluster namespace.

    ```yaml
    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: ManagedClusterAddOn
    metadata:
      name: hypershift-addon
      namespace: local-cluster
    spec:
      installNamespace: open-cluster-management-agent-addon
    ```

    Validation:
    ```bash
    $ oc get ManagedClusterAddOn -n local-cluster hypershift-addon
    NAME               AVAILABLE   DEGRADED   PROGRESSING
    hypershift-addon   True
    $ oc get pods -n hypershift
    NAME                        READY   STATUS    RESTARTS   AGE
    operator-7dbdcc4b49-4dzsq   1/1     Running   0          2m18s
    ```

7. Create a cloud provider secret to grant access to your AWS subscription.  The credentials used must have access to create infrastructure resources like VPCs, subnets, NAT Gateways, etc.

    ```yaml
    apiVersion: v1
    metadata:
      name: my-aws-cred
      namespace: default      # Where you create HypershiftDeployment resources
    type: Opaque
    kind: Secret
    stringData:
      ssh-publickey:          # Value
      ssh-privatekey:         # Value
      pullSecret:             # Value, required
      baseDomain:             # Value, required
      aws_secret_access_key:  # Value, required
      aws_access_key_id:      # Value, required
    ```

    Q3:  Any limitations or implications if HyperShift HUB and HyperShift managed clusters be on different accounts? Different clouds providers?

8. Create HypershiftDeployment object
    ```yaml
    apiVersion: cluster.open-cluster-management.io/v1alpha1
    kind: HypershiftDeployment
    metadata:
      name: <cluster>
      namespace: default
    spec:
      hostingCluster: <hosting-service-cluster>
      hostingNamespace: clusters
      hostedClusterSpec:
        networking:
          machineCIDR: 10.0.0.0/16    # Default
          networkType: OpenShiftSDN
          podCIDR: 10.132.0.0/14      # Default
          serviceCIDR: 172.31.0.0/16  # Default
        platform:
          type: AWS
        pullSecret:
          name: <cluster>-pull-secret    # This secret is created by the controller
        release:
          image: quay.io/openshift-release-dev/ocp-release:4.10.15-x86_64  # Default
        services:
        - service: APIServer
          servicePublishingStrategy:
            type: LoadBalancer
        - service: OAuthServer
          servicePublishingStrategy:
            type: Route
        - service: Konnectivity
          servicePublishingStrategy:
            type: Route
        - service: Ignition
          servicePublishingStrategy:
            type: Route
        sshKey: {}
      nodePools:
      - name: <cluster>
        spec:
          clusterName: <cluster>
          management:
            autoRepair: false
            replace:
              rollingUpdate:
                maxSurge: 1
                maxUnavailable: 0
              strategy: RollingUpdate
            upgradeType: Replace
          platform:
            aws:
              instanceType: m5.large
            type: AWS
          release:
            image: quay.io/openshift-release-dev/ocp-release:4.10.15-x86_64 # Default
          replicas: 2
      infrastructure:
        cloudProvider:
          name: <my-secret>
        configure: True
        platform:
          aws:
            region: <region>
    ```

    Q4: How do I set the CIDRs to different values?  I've tried [multiple combinations](https://github.com/ncolon/hypershift-exploration/blob/main/04-hypershift-c2.yaml#L14-L23) of `machineCIDR` and `machineNetwork` with no effect on provisioned IP addresses of worker nodes.

    Q5: How to deploy a HypershiftDeployment in multizone by default. Setting [`infrastructureAvailabilityPolicy: HighlyAvailable`](https://github.com/ncolon/hypershift-exploration/blob/main/04-hypershift-c2.yaml#L10) does create a multizone VPC, but initial nodepool only deployed to a single zone.

    Q6: How to deploy a HA hostedControlPlane?  By default a `HypershiftDeployment` will deploy a `HostedControlPlane` with single replicas of control plane operators by default.

    A6: Set [`controllerAvailabilityPolicy: HighlyAvailable`](https://github.com/ncolon/hypershift-exploration/blob/main/04-hypershift-c2.yaml#L11)

    Q7: What is the recommended way of manually scaling a hostedCluster?

    - Scaling MachineSet.cluster.x-k8s.io does not work
    - Scaling NodePool.hypershift.openshift.io does not work
    - Manually editing HypershiftDeployment [NodePool replicas](https://github.com/ncolon/hypershift-exploration/blob/main/04-hypershift-c2.yaml#L47) works.

    Is there a declarative way of doing so?
    
    ```bash
    $ oc get machinesets -A                                                           
    NAMESPACE                NAME                       CLUSTER               REPLICAS   READY   AVAILABLE   AGE   VERSION
    clusters-hypershift-c1   hypershift-c1-55cb54d9f8   hypershift-c1-p4zf8   2          2       2           22m   4.10.15
    $ oc scale machinesets -n clusters-hypershift-c1   hypershift-c1-55cb54d9f8 --replicas 3          
    machineset.cluster.x-k8s.io/hypershift-c1-55cb54d9f8 scaled
    $ oc get machinesets -A              
    NAMESPACE                NAME                       CLUSTER               REPLICAS   READY   AVAILABLE   AGE   VERSION
    clusters-hypershift-c1   hypershift-c1-55cb54d9f8   hypershift-c1-p4zf8   3          2       2           27m   4.10.15
    $ oc scale nodepool -n clusters hypershift-c1 --replicas 3                              
    nodepool.hypershift.openshift.io/hypershift-c1 scaled
    $ oc get nodepools,machinesets -A                         
    NAMESPACE   NAME                                             CLUSTER         DESIRED NODES   CURRENT NODES   AUTOSCALING   AUTOREPAIR   VERSION   UPDATINGVERSION   UPDATINGCONFIG   MESSAGE
    clusters    nodepool.hypershift.openshift.io/hypershift-c1   hypershift-c1   2               2               False         False        4.10.15                                      

    NAMESPACE                NAME                                                   CLUSTER               REPLICAS   READY   AVAILABLE   AGE   VERSION
    clusters-hypershift-c1   machineset.cluster.x-k8s.io/hypershift-c1-55cb54d9f8   hypershift-c1-p4zf8   2          2       2           29m   4.10.15

    $ oc get hd -A                   
    NAMESPACE   NAME            TYPE   INFRA                  IAM                    MANIFESTWORK           PROVIDER REF   PROGRESS    AVAILABLE
    default     hypershift-c1   AWS    ConfiguredAsExpected   ConfiguredAsExpected   ConfiguredAsExpected   AsExpected     Completed   True
    $ oc edit hd -n default hypershift-c1 # << THIS WORKS
    ```

    Q8: Is there an easier way to add nodePools to an existing multizone cluster?  [This works](https://github.com/ncolon/hypershift-exploration/blob/main/05-hypershift-c2-storage-nodepool.yaml) but the process is completely manual.  I had to `oc get nodepool hypershift-c2 -o yaml` build a nodepool for each zone, all while digging thru the AWS console for the right subnet-id for each zone.