---
title: Set up persistent storage 
badge: <i class='ent'/></i>
type: guide
order: 206
meta_title: Set up persistent storage with Label Studio Enterprise
meta_description: Configure persistent storage with Label Studio Enterprise hosted in the cloud to store uploaded data such as task data, user images, and more. 
---

If you host Label Studio Enterprise in the cloud, you want to set up persistent storage for uploaded task data, user images, and more in the same cloud service as your deployment.

Follow the steps relevant for your deployment:
* [Set up Amazon S3](#Set-up-Amazon-S3) for Label Studio Enterprise deployments in Amazon Web Services (AWS).
* [Set up Google Cloud Storage (GCS)](#Set-up-Google-Cloud-Storage) for Label Studio Enterprise deployments in Google Cloud Platform.
* [Set up Microsoft Azure Storage](#Set-up-Microsoft-Azure-Storage) for Label Studio Enterprise deployments in Microsoft Azure. 

## Set up Amazon S3

Set up Amazon S3 as the persistent storage for Label Studio Enterprise hosted in AWS. 

### Create an S3 bucket

Start by [creating an S3 bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/create-bucket-overview.html) following the Amazon Simple Storage Service User Guide steps.

> If you want to secure the data stored in the S3 bucket at rest, you can [set up default server-side encryption for Amazon S3 buckets](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-encryption.html) following the steps in the Amazon Simple Storage Service User Guide. 

### Configure the S3 bucket
After you create an S3 bucket, set up the necessary IAM permissions to grant Label Studio Enterprise access to your bucket. There are three ways that you can manage access to your S3 bucket:
- Set up an **IAM role** with an OIDC provider (**recommended**)
- Use **access keys**
- Set up an **IAM role** without an OIDC provider

Select the relevant tab and follow the steps for your desired option: 

<div class="code-tabs">
  <div data-name="IAM role (OIDC)">

> To set up an IAM role using this method, you must have a configured and provisioned OIDC provider for your cluster. See [Create an IAM OIDC provider for your cluster](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html) in the Amazon EKS User Guide.

1. Follow the steps to [create an IAM role and policy for your service account](https://docs.aws.amazon.com/eks/latest/userguide/create-service-account-iam-policy-and-role.html) in the Amazon EKS User Guide.
2. Use the following IAM Policy, replacing `<YOUR_S3_BUCKET>` with the name of your bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::<YOUR_S3_BUCKET>"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::<YOUR_S3_BUCKET>/*"
      ]
    }
  ]
}
```

3. Create an **IAM role as a Web Identity** using the cluster OIDC provider as the identity provider:
   1. Create a new **Role** from your IAM Console.
   2. Select the **Web identity** Tab.
   3. In the **Identity Provider** drop-down, select the OpenID Connect provider URL of your EKS and `sts.amazonaws.com` as the Audience.
   4. Attach the newly created permission to the Role and name it.
   5. Retrieve the Role arn for the next step.
4. After you create an IAM role, add it as an annotation in your `lse-values.yaml` file:

```yaml
global:
  persistence:
    enabled: true
    type: s3
    config:
      s3:
        bucket: "<YOUR_BUCKET_NAME>"
        region: "<YOUR_BUCKET_REGION>"
app:
  serviceAccount:
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME_FROM_STEP_3>

rqworker:
  serviceAccount:
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME_FROM_STEP_3>
```

  </div>

  <div data-name="Access keys">

1. Create an IAM user with **Programmatic access**. See [Creating an IAM user in your AWS account](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) in the AWS Identity and Access Management User Guide. 
2. When creating the user, for the **Set permissions** option, choose to **Attach existing policies directly**.
3. Select **Create policy** and attach the following policy, replacing `<YOUR_S3_BUCKET>` with the name of your bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::<YOUR_S3_BUCKET>"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::<YOUR_S3_BUCKET>/*"
      ]
    }
  ]
}
```

4. After you create the user, save the username and access key somewhere secure.
5. Update your `lse-values.yaml` file with your newly-created access key ID and secret key as `<YOUR_ACCESS_KEY_ID>` and `<YOUR_SECRET_ACCESS_KEY>`:

```yaml
global:
  persistence:
    enabled: true
    type: s3
    config:
      s3:
        accessKey: "<YOUR_ACCESS_KEY_ID>"
        secretKey: "<YOUR_SECRET_ACCESS_KEY>"
        bucket: "<YOUR_BUCKET_NAME>"
        region: "<YOUR_BUCKET_REGION>"
```

  </div>

  <div data-name="IAM role (EKS node)">

To create an IAM role without using OIDC in EKS, follow these steps.

1. In the AWS console UI, go to **EKS > Clusters > `YOUR_CLUSTER_NAME` > Node Group**.
2. Select the name of `YOUR_NODE_GROUP` with Label Studio Enterprise deployed.
3. On the **Details** page, locate and select the option for Node IAM Role ARN and choose to **Attach existing policies directly**.
3. Select **Create policy** and attach the following policy, replacing `<YOUR_S3_BUCKET>` with the name of your bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::<YOUR_S3_BUCKET>"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::<YOUR_S3_BUCKET>/*"
      ]
    }
  ]
}
```

4. After you add an IAM policy, configure your `lse-values.yaml` file:

```yaml
global:
  persistence:
    enabled: true
    type: s3
    config:
      s3:
        bucket: "<YOUR_BUCKET_NAME>"
        region: "<YOUR_BUCKET_REGION>"
```

  </div>
</div>

## Set up Google Cloud Storage

Set up Google Cloud Storage (GCS) as the persistent storage for Label Studio Enterprise hosted in Google Cloud Platform (GCP).

### Create a GCS bucket 

1. Start by creating a bucket. See [Creating storage buckets](https://cloud.google.com/storage/docs/creating-buckets) in the Google Cloud Storage guide. For example, a bucket called `heartex-example-bucket-123456`.
2. When choosing the [access control method for the bucket](https://cloud.google.com/storage/docs/access-control), choose **uniform access control**. 
3. Create an IAM Service Account. See [Creating and managing service accounts](https://cloud.google.com/iam/docs/creating-managing-service-accounts) in the Google Cloud Storage guide. 
4. Select the predefined **Storage Object Admin** IAM role to add to the service account so that the account can create, access, and delete objects in the bucket.
5. Add a condition to the role that restricts the role to access only objects that belong to the bucket you created. You can add a condition in one of two ways:
    - Select **Add Condition** when setting up the service account IAM role, then use the **Condition Builder** to specify the following values:
      - Condition type: `Name`
      - Operator: `Starts with`
      - Value: `projects/_/buckets/heartex-example-bucket-123456`
    - Or, **use a Common Expression Language** (CEL) to specify an IAM condition. For example, set the following: `resource.name.startsWith('projects/_/buckets/heartex-example-bucket-123456')`. See [CEL for Conditions in Overview of IAM Conditions](https://cloud.google.com/iam/docs/conditions-overview#cel) in the Google Cloud Storage guide. 

### Configure to use GCS bucket

You can connect Label Studio Enterprise to your GCS bucket using **Workload Identity** or **Access keys**.

After you create a bucket and set up IAM permissions, connect Label Studio Enterprise to your GCS bucket. There are two ways that you can connect to your bucket:
- Use Workload Identity to allow workloads in GKE to access your GCS bucket by impersonating the service account you created (**recommended**).
- Create a service account key to use the service account outside Google Cloud.  

<div class="code-tabs">
<div data-name="Workload Identity">

> Make sure that Workload Identity is enabled on your GKE cluster and that you meet the necessary prerequisites. See [Using Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) in the Google Kubernetes Engine guide.

1. Set up the following environment variables, specifying the service account you created as the `GCP_SA` variable, and replacing the other references in `<>` as needed:

```shell
GCP_SA=<Service-Account-You-Created>
APP_SA="serviceAccount:<GCP_PROJECT_ID>.svc.id.goog[<K8S_NAMESPACE>/<HELM_RELEASE_NAME>-lse-app]"
WORKER_SA="serviceAccount:<GCP_PROJECT_ID>.svc.id.goog[<K8S_NAMESPACE>/<HELM_RELEASE_NAME>-lse-rqworker]"
```

2. Create an IAM policy binding between the Kubernetes service account on your cluster and the GCS service account you created, allowing the K8s service account for the Label Studio Enterprise app and the related rqworkers to impersonate the other service account. From the command line, run the following:

```shell
gcloud iam service-accounts add-iam-policy-binding ${GCP_SA} \
    --role roles/iam.workloadIdentityUser \
    --member "${APP_SA}"
gcloud iam service-accounts add-iam-policy-binding ${GCP_SA} \
    --role roles/iam.workloadIdentityUser \
    --member "${WORKER_SA}"
```

3. After binding the service accounts, update your `lse-values.yaml` file to include the values for the service account and other configurations. Update the `projectID`, `bucket`, and replace the`<GCP_SERVICE_ACCOUNT>` with the relevant values for your deployment:

```yaml
global:
  persistence:
    enabled: true
    type: gcs
    config:
      gcs:
        projectID: "<YOUR_PROJECT_ID>"
        bucket: "<YOUR_BUCKET_NAME>"
app:
  serviceAccount:
    annotations:
      iam.gke.io/gcp-service-account: "<GCP_SERVICE_ACCOUNT>"

rqworker:
  serviceAccount:
    annotations:
      iam.gke.io/gcp-service-account: "<GCP_SERVICE_ACCOUNT>"
```

</div>
<div data-name="Service Account Key">

1. Create a service account key from the UI and download the JSON. Follow the steps for [Creating and managing service account keys](https://cloud.google.com/iam/docs/creating-managing-service-account-keys) in the Google Cloud Identity and Access Management guide.
2. After downloading the JSON for the service account key, update or create references to the JSON, your projectID, and your bucket in your `lse-values.yaml` file:

```yaml
global:
  persistence:
    enabled: true
    type: gcs
    config:
      gcs:
        projectID: "<YOUR_PROJECT_ID>"
        applicationCredentialsJSON: "<YOUR_JSON>"
        bucket: "<YOUR_BUCKET_NAME>"
```

  </div>
</div>


## Set up Microsoft Azure Storage

Create a Microsoft Azure Storage container to use as persistent storage with Label Studio Enterprise.
### Create a Storage container

1. Create an Azure storage account: https://docs.microsoft.com/en-us/azure/storage/common/storage-account-overview
> Make sure that you set Stock Keeping Unit (SKU) to Premium_LRS and the kind parameter is set to BlockBlobStorage. This configuration results in a storage that uses SSDs rather than standard Hard Disk Drives (HDD). If you set this parameter to an HDD-based storage option, your instance may be too slow and might malfunction.
2. Find the generated key in the **Storage accounts > Access keys section** in the [Azure Portal](https://portal.azure.com/) or by running the following command:

```shell
az storage account keys list --account-name=${STORAGE_ACCOUNT}
```

3. Create a new storage container within your storage account by following the [official documentation](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-portal) or by running the following command:

```shell
az storage container create --name <YOUR_CONTAINER_NAME> \
          --account-name <YOUR_STORAGE_ACCOUNT> \
          --account-key "<YOUR_STORAGE_KEY>"
```

### Configure to use Azure container
Update your `lse-values.yaml` file with the `YOUR_CONTAINER_NAME`, `YOUR_STORAGE_ACCOUNT`, and `YOUR_STORAGE_KEY` that you created:

```yaml
global:
  persistence:
    enabled: true
    type: azure
    config:
      azure:
        storageAccountName: "<YOUR_STORAGE_ACCOUNT>"
        storageAccountKey: "<YOUR_STORAGE_KEY>"
        containerName: "<YOUR_CONTAINER_NAME>"
```
