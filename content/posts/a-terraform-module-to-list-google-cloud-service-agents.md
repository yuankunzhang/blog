---
title: "A Terraform Module to List Google Cloud Service Agents"
date: 2020-04-01T23:47:03+08:00
draft: false
toc: false
images:
tags:
  - gcp
  - terraform
---

Google Cloud Platform presents [two distinct types of service accounts](https://cloud.google.com/iam/docs/service-accounts#types_of_service_accounts): user-managed service accounts and Google-managed service accounts. The former is typically employed by user applications as an interface with Google Cloud, whereas the latter is utilized internally by Google Cloud. Within the realm of Google-managed service accounts, a specialized subset exists: Google Cloud Service Agents. These service agents are used by Google Cloud services to operate internal processes necessary to fulfill user-requested operations.

A service agent adhers to the following template:

```
service-PROJECT_NUMBER@SERVICE_NAME.iam.gserviceaccount.com
```

These service agents are easily identifiable within the IAM section of the Google Cloud Console.

![Service Agents](/img/20200402-0012.png)

During the management of IAM binding policies via Terraform, these service agents can often become obtrusive. For illustration, consider the following code snippet from one of our Terraform files (where `xxxxx` substitutes the actual project number).

<!--more-->

```hcl
data "google_project" "project" {}

resource "google_project_iam_binding" "cloudbuild_service_agent" {
  project = data.google_project.project.project_id
  role    = "roles/cloudbuild.serviceAgent"

  members = [
    "serviceAccount:service-xxxxx@gcp-sa-cloudbuild.iam.gserviceaccount.com",
  ]
}

resource "google_project_iam_binding" "container_service_agent" {
  project = data.google_project.project.project_id
  role    = "roles/container.serviceAgent"

  members = [
    "serviceAccount:service-xxxxx@container-engine-robot.iam.gserviceaccount.com",
  ]
}
```

The complication intensifies when referencing service agents from different projects, as these project numbers can often resemble what we term "magic numbers".

To address this issue, I've developed a simple Terraform module dubbed [google-cloud-service-agents](https://github.com/yuankunzhang/google-cloud-service-agents). This module accepts a project ID as input and provides a list of service agents as output. By leveraging this module, the aforementioned code snippet can be effectively refactored as follow:

```hcl
data "google_project" "project" {}

module "agents" {
  source = "/path/to/module"

  project_number = data.google_project.project.number
}

resource "google_project_iam_binding" "cloudbuild_service_agent" {
  project = data.google_project.project.project_id
  role    = "roles/cloudbuild.serviceAgent"

  members = [
    "serviceAccount:${module.agents.cloud_build}",
  ]
}

resource "google_project_iam_binding" "container_service_agent" {
  project = data.google_project.project.project_id
  role    = "roles/container.serviceAgent"

  members = [
    "serviceAccount:${module.agents.container_engine}",
  ]
}
```

For more information, kindly refer to the [Github repository](https://github.com/yuankunzhang/google-cloud-service-agents).
