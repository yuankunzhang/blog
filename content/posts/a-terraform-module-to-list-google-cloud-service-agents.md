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

There are [two types of service accounts](https://cloud.google.com/iam/docs/service-accounts#types_of_service_accounts) in Google Cloud: user-managed service accounts, which are used by user applications to talk to Google Cloud; and Google-managed services accounts, which are used by Google Cloud internally. Among the second category, there is a special subtype of service accounts called Google Cloud Service Agents. Service Agents are used by Google Cloud services to run internal processes so that user requested operations can be fulfilled.

A service agent has the following pattern:

```
service-PROJECT_NUMBER@SERVICE_NAME.iam.gserviceaccount.com
```

You can spot the service agents from the IAM section of Google Cloud Console.

![Service Agents](/img/20200402-0012.png)

When managing IAM binding policies via Terraform, these service agents often generate noises. As an example, I'll show you a code snippet coming from one of our Terraform files (I'm using `xxxxx` instead of the real project number).

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

When you are refering to service agents from other projects, things get even worse. Those project numbers look very much lick what we call "the magic number".

To tackle this, I wrote a simple Terraform module named [google-cloud-service-agents](https://github.com/yuankunzhang/google-cloud-service-agents). It consumes a project ID and exposes a list of service agents. With this module, the above code snippet can be rewritten to this:

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

For more information, please check out the [Github repository](https://github.com/yuankunzhang/google-cloud-service-agents).
