# gke-workload-identity-demo

### Deploy The Cluster
0. Enable APIs
    ```
    gcloud services enable container.googleapis.com \
                           redis.googleapis.com \
                           cloudresourcemanager.googleapis.com \
                           servicenetworking.googleapis.com \
                           cloudbuild.googleapis.com
    ```
0. Open the GKE console and create a cluster named `cluster-1` with the following settings:
    * Set **Cluster basics > Location Type** as **Regional**
    * Turn on **Security > Enable Workload Identity**
    * Turn on **Features > Enable Config Connector**

### Configure Config Connector
0. Create the cluster
    ```
    gcloud container clusters get-credentials cluster-1 --region us-central1
    ```
0. Create the service account config connector will use
    ```
    gcloud iam service-accounts create sample-app-config-connector
    ```
0. Give the config connector service account the project owner role so it can create and configure resources in your project
    ```
    gcloud projects add-iam-policy-binding $(gcloud config get-value project) \
        --member="serviceAccount:sample-app-config-connector@$(gcloud config get-value project).iam.gserviceaccount.com" \
        --role="roles/owner"
    ```
0. Bind the config connector **Google Cloud** service account to the config connector **Kubernetes** service account using the workload identity role
    ```
    gcloud iam service-accounts add-iam-policy-binding \
        sample-app-config-connector@$(gcloud config get-value project).iam.gserviceaccount.com \
        --member="serviceAccount:$(gcloud config get-value project).svc.id.goog[cnrm-system/cnrm-controller-manager]" \
        --role="roles/iam.workloadIdentityUser"
    ```
0. Configure config connector kubernetes resource with the Google Cloud service account
    ```
    sed -i s/PROJECT_ID/$(gcloud config get-value project)/g ./config-connector.yaml
    kubectl apply -f config-connector.yaml --context gke_$(gcloud config get-value project)_us-central1_cluster-1
    ```
0. Tell config connector which project to deploy to when resources are created in the default namespace
    ```
    kubectl annotate namespace default cnrm.cloud.google.com/project-id=$(gcloud config get-value project) --context gke_$(gcloud config get-value project)_us-central1_cluster-1
    ```
0. Create a network for pods to use to communicate with their redis instance
    ```
    kubectl apply -f default-network.yaml --context gke_$(gcloud config get-value project)_us-central1_cluster-1
    gcloud compute addresses create sample-app \
        --global \
        --purpose=VPC_PEERING \
        --prefix-length=16 \
        --description="Sample App range" \
        --network=default
    gcloud services vpc-peerings connect \
        --service=servicenetworking.googleapis.com \
        --ranges=sample-app \
        --network=default \
        --project=$(gcloud config get-value project)
    ```

### Deploy The Application
0. Build the container image
    ```
    gcloud builds submit . --tag=gcr.io/$(gcloud config get-value project)/sample-app
    ```
0. Update the deployment
    ```
    sed -i s/PROJECT_ID/$(gcloud config get-value project)/g k8s/base/deployments/*
    ```
0. Deploy the application and redis
    ```
    kubectl apply -k ./k8s/prod
    ```
