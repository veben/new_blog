---
title: "Mario Bros game, deployed on Kubernetes cluster, managed by Civo, created with Terraform"
date: 2024-05-30
tags:
  - kubernetes
  - helm
  - civo
  - terraform
---

In previous posts, we explored [deploying a Kubernetes cluster with Civo](https://veben.github.io/civo-cluster/) and [creating a Helm chart for the Mario Bros game](https://veben.github.io/mario-helm/). Now, let's combine these steps and deploy the Mario Bros **Helm** chart on a Kubernetes cluster managed by **Civo** using **Terraform**.

The associated code is available [here](https://github.com/veben/civo_tf_k8s_mario) 

## Prerequisites
Before we start, ensure you have the following:
- A **Civo account** with an access key.
- **Terraform** installed on your local machine.
- **kubectl** installed on your local machine.

## Overview of the repository
Here's a high-level view of the repository structure:

```plaintext
civo_tf_k8s_mario/
├── providers.tf
├── variables.tf
├── outputs.tf
├── cluster_firewall.tf
└── main.tf
```

Each of these files has a specific purpose  in the Terraform configuration:
- `providers.tf`: Configures the necessary providers.
- `variables.tf`: Defines the input variables for the Terraform script.
- `outputs.tf`: Specifies output values.
- `cluster_firewall.tf`: Manages the firewall rules for the cluster.
- `main.tf`: Contains the main configuration for the Kubernetes cluster and Helm chart.

## Configuration
### 1. Variables
Create a `terraform.tfvars` file where you can tailor the cluster according to your requirements. The two most crucial variables to customize are the **region** where you want to deploy the infrastructure and the **cluster_node_size**.
Note that it contains a variable `api_key` that can be defined here or passed to Terraform via environment variable.
```plaintext
api_key               = "your-civo-api-key"
region                = "FRA1"
cluster_name          = "my-mario-civo-cluster"
cluster_node_size     = "g4s.kube.xsmall"
cluster_node_count    = 1
kubernetes_api_access = ["0.0.0.0/0"]
kube_config_path      = "~/.kube/my_mario_k8s_config.yaml"
chart_name            = "mario-bros"
chart_version         = "0.4.0"
chart_repository      = "https://veben.github.io/helm_charts/"
```

The `variables.tf` file contains all the customizable parameters with default values and validation rules.
```hcl
variable "api_key" {
  description = "API key for authentication."
  type        = string
  sensitive   = true
  validation {
    condition     = length(var.api_key) > 0
    error_message = "The API key cannot be empty."
  }
}
variable "region" {
  description = "The region to provision the cluster in (e.g., FRA1 for Frankfurt)."
  type        = string
  default     = "FRA1"
  validation {
    condition     = contains(["FRA1", "NYC1", "LON1", "SFO1"], var.region)
    error_message = "The specified region is not supported. Please choose from 'FRA1', 'NYC1', 'LON1', 'SFO1'."
  }
}
variable "cluster_name" {
  description = "The name of the Kubernetes cluster to be created."
  type        = string
  default     = "my-mario-civo-cluster"
  validation {
    condition     = length(var.cluster_name) > 0
    error_message = "The cluster name cannot be empty."
  }
}

variable "cluster_node_size" {
  description = "The size of the nodes to provision. Run `civo size list` for all options."
  type        = string
  default     = "g4s.kube.xsmall"
  validation {
    condition     = contains(["g4s.kube.xsmall", "g4s.kube.small", "g4s.kube.medium"], var.cluster_node_size)
    error_message = "The specified node size is not supported. Please choose from 'g4s.kube.xsmall', 'g4s.kube.small', 'g4s.kube.medium'."
  }
}

variable "cluster_node_count" {
  description = "The number of nodes in the default pool."
  type        = number
  default     = 1
  validation {
    condition     = var.cluster_node_count > 0
    error_message = "The number of nodes must be greater than zero."
  }
}

variable "kubernetes_api_access" {
  description = "List of subnets allowed to access the Kubernetes API."
  type        = list(string)
  default     = ["0.0.0.0/0"]
  validation {
    condition     = alltrue([for subnet in var.kubernetes_api_access : can(regex("^(\\d{1,3}\\.){3}\\d{1,3}/\\d{1,2}$", subnet))])
    error_message = "Each entry in kubernetes_api_access must be a valid subnet in CIDR notation."
  }
}
variable "kube_config_path" {
  description = "The path to save the Kubernetes configuration file."
  type        = string
  default     = "~/.kube/civo_tf_k8s_mario_config.yaml"
}

variable "chart_name" {
  description = "The name of the Helm chart to be deployed."
  type        = string
  default     = "mario-bros"
}

variable "chart_version" {
  description = "The version of the Helm chart to be deployed."
  type        = string
  default     = "0.4.0"
}
variable "chart_repository" {
  description = "The repository URL of the Helm chart."
  type        = string
  default     = "https://veben.github.io/helm_charts/"
}
```

### 2. Providers
In `providers.tf`, configure the Civo and Helm providers:
```hcl
terraform {
  required_providers {
    civo = {
      source  = "civo/civo"
      version = "1.0.41"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "2.13.2"
    }
  }
}

provider "civo" {
  token  = var.api_key
  region = var.region
}

provider "helm" {
  kubernetes {
    config_path = var.kube_config_path
  }
}
```

### 3. Cluster Firewall
In `cluster_firewall.tf`, define the firewall rule to allow access to the Kubernetes API server (default port is `6443`):
```hcl
resource "civo_firewall" "firewall" {
  name                 = "${var.cluster_name}-firewall"
  create_default_rules = false

  ingress_rule {
    protocol   = "tcp"
    port_range = "6443"
    cidr       = var.kubernetes_api_access
    label      = "kubernetes-api-server"
    action     = "allow"
  }
}
```

### 4. Main
In `main.tf`, define the Civo Kubernetes cluster and the Helm chart deployment:

```hcl
resource "civo_kubernetes_cluster" "cluster" {
  name        = var.cluster_name
  firewall_id = civo_firewall.firewall.id

  pools {
    node_count = var.cluster_node_count
    size       = var.cluster_node_size
  }

  timeouts {
    create = "5m"
  }
}

resource "local_file" "cluster-config" {
  content              = civo_kubernetes_cluster.cluster.kubeconfig
  filename             = pathexpand(var.kube_config_path)
  file_permission      = "0600"
  directory_permission = "0755"
}

resource "helm_release" "mario" {
  name       = var.chart_name
  repository = var.chart_repository
  chart      = var.chart_name
  version    = var.chart_version

  depends_on = [local_file.cluster-config]
}
```

### 5. Output
In `outputs.tf`, define the output for the Kubernetes configuration file path, in order to be able to configure `kubectl` to access to it.
```hcl
output "kubeconfig_path" {
  value = local_file.cluster-config.filename
}
```

## Deploying
- If you have not defined it in the `terraform.tfvars` file, save the Civo API key as an environment variable (replace **** with your actual Civo API key):
```sh
export TF_VAR_api_key=****
```
- Initialize Terraform, plan the deployment, and apply the configurations:
```sh
terraform init
terraform plan
terraform apply --auto-approve
```

## Testing
- Configure `KUBECONFIG` environment variable to point to the Kubernetes configuration file:
```sh
export KUBECONFIG=`terraform output -raw kubeconfig_path`
```
- Display the URL of the game:
```sh
echo http://`kubectl get service | grep mario-bros | awk '{print $4}'`:80
```
- Copy the displayed URL and paste it into your web browser to play the game.

## Cleaning up
To clean up the resources, run:
```sh
terraform destroy --auto-approve
```