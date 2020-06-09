# eks-terraform-spot-autoscaler

## Introduction

In this lab, we use demonstrate how terraform (instead of eksctl/cloudformation) can be used to manage EKS cluster and its nodegroups. Following are components in this lab:

1. Create a cluster
1. Create a spot nodegroup
1. Deploy sample application
1. Create a horizontal pod autoscaler (HPA)
1. Create a cluster autoscaler (CA)
1. Simulate a load test to scale both the pods and then the nodes

## Assumptions

1. Follow the instructions from <eksworkshop.com> to set up Cloud9 environment
1. Install kubectl

## Steps

1. Follow the instructions from <https://learn.hashicorp.com/terraform/kubernetes/provision-eks-cluster> to
   * Install terraform (follow the wget option if you are using Cloud9)

   ```bash
   wget https://releases.hashicorp.com/terraform/0.12.26/terraform_0.12.26_linux_amd64.zip
   unzip terraform_0.12.26_linux_amd64.zip
   sudo mv terraform /usr/local/bin/
   terraform -install-autocomplete
   exec bash
   echo 'alias tf=terraform' >> ~/.bashrc
   ```

   * Clone the sample templates

   ```bash
   git clone https://github.com/hashicorp/learn-terraform-provision-eks-cluster
   ```

   * Review the templates

   *vpc.tf* provisions a VPC, subnets and availability zones using the AWS VPC Module. A new VPC is created for this guide so it doesn't impact your existing cloud environment and resources.

   *security-groups.tf* provisions the security groups used by the EKS cluster.

   *eks-cluster.tf* provisions all the resources (AutoScaling Groups, etc...) required to set up an EKS cluster in the private subnets and bastion servers to access the cluster using the AWS EKS Module.

   *outputs.tf* defines the output configuration.

   *versions.tf* sets the Terraform version to at least 0.12. It also sets versions for the providers used in this sample.

   * Modify the eks-cluster.tf template removing the worker groups and add a new spot worker group. Optionally, you can modify the cluster name and other parameters of your choice.

   ```bash
   worker_groups_launch_template = [
    {
      name                    = "spot-launchteplate-1"
      override_instance_types = ["t3.medium", "t3.large"]
      spot_instance_pools     = 2 # how many spot pools per az, len matches instances types len
      asg_max_size            = 8
      kubelet_extra_args      = "--node-labels=node.kubernetes.io/lifecycle=spot"
      protect_from_scale_in   = true # let cluster autoscaler handle
    },
   ]
   ```

   This creates an autoscaling group (ASG) which provision fixed number of instances.

   * Initialize Terraform workspace

   ```bash
   tf init
   ```

   * Provision the cluster

   ```bash
   tf apply
   ```

   Capture the cluster's name. If you miss it, run `tf output`. If you make changes to the cluster, run `tf plan` and `tf apply --auto-approve`.

   * Configure kubectl

   ```bash
   aws eks --region us-west-2 update-kubeconfig --name <cluster name>
   ```

1. Follow this [section](https://eksworkshop.com/beginner/080_scaling/) to set up both HPA and CA.

   * On HPA configurations, change the cpu threshold to 10% instead of 50%, and maximum pods to 50 instead of 10. You may skip running the busybox invocations until after CA setup.
   * *$ROLE_NAME* can be retrieved from the AWS EC2 console instead.

1. Run tests

   * Open 3 terminals

   ```bash
   kubectl get pods -o wide -w
   kubectl get nodes -w
   kubectl logs -f deployment/cluster-autoscaler -n kube-system
   ```

   * Open another terminal and run the busybox test

   ```bash
   while true; do wget -q -O - http://php-apache; done
   ```

1. Verify that pods are scaled up following with nodes scaled up.
1. Terminate the busybox test and observe both pods and nodes are scaled down.
1. Clean up following this [section](https://eksworkshop.com/beginner/080_scaling/cleanup/)
1. Clean up the cluster

   ```bash
   tf destroy
   ```

## Refs

1. <https://learn.hashicorp.com/terraform/kubernetes/provision-eks-cluster>
