# gke-workload-identity-demo
0. `gcloud services enable container.googleapis.com`
0. Open the GKE console and create a cluster with the following settings:
    * Turn on **Security > Enable Workload Identity**
    * Turn on **Features > Enable Config Connector**

### TODO:
0. Deploy the application
0. Configure Config Connector
    ```
    gcloud iam service-accounts create sample-app-config-connector
    gcloud projects add-iam-policy-binding $(gcloud config get-value project) \
        --member="serviceAccount:sample-app-config-connector@$(gcloud config get-value project).iam.gserviceaccount.com" \
        --role="roles/owner"
    gcloud iam service-accounts add-iam-policy-binding \
        sample-app-config-connector@$(gcloud config get-value project).iam.gserviceaccount.com \
        --member="serviceAccount:$(gcloud config get-value project).svc.id.goog[cnrm-system/cnrm-controller-manager]" \
        --role="roles/iam.workloadIdentityUser"
    
    cat > config-connector.yaml <<EOF
    apiVersion: core.cnrm.cloud.google.com/v1beta1
    kind: ConfigConnector
    metadata:
      name: configconnector.core.cnrm.cloud.google.com
    spec:
     mode: cluster
     googleServiceAccount: "sample-app-config-connector@$(gcloud config get-value project).iam.gserviceaccount.com"
    EOF
    kubectl apply -f config-connector.yaml --context gke_$(gcloud config get-value project)_us-central1_staging
    kubectl annotate namespace default cnrm.cloud.google.com/project-id=$(gcloud config get-value project) \
                    --context gke_$(gcloud config get-value project)_us-central1_staging
    kubectl apply -f config-connector.yaml --context gke_$(gcloud config get-value project)_us-central1_prod
    kubectl annotate namespace default cnrm.cloud.google.com/project-id=$(gcloud config get-value project) \
                    --context gke_$(gcloud config get-value project)_us-central1_prod
    # Create default network in both clusters so it can be referenced
    cat > default-network.yaml <<EOF
    ---
    apiVersion: compute.cnrm.cloud.google.com/v1beta1
    kind: ComputeNetwork
    metadata:
      name: default
    spec:
      routingMode: REGIONAL
      autoCreateSubnetworks: true
    EOF
    kubectl apply -f default-network.yaml --context gke_$(gcloud config get-value project)_us-central1_staging
    kubectl apply -f default-network.yaml --context gke_$(gcloud config get-value project)_us-central1_prod
    ```