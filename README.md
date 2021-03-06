# EKS-specific Kubernetes Drone Plugin

This Drone plugin allows you to deploy new container images to an EKS-based
Kubernetes cluster. It makes some assumptions about being exclusively AWS, such
as using the EKS API to discover the cluster endpoint and CA certificate and
using IAM to authenticate the role.

The advantage of this method is that you don't need to populate Drone with
secrets and can take advantage of the IAM security model.

Inspired by the work in https://github.com/honestbee/drone-kubernetes which is
a great option for any non-EKS K8 environments.

# Usage (in deployment mode)

| Argument        | Required? | Details                                                                                         |
|-----------------|-----------|-------------------------------------------------------------------------------------------------|
| IAM_ROLE_ARN    | required  | ARN for an IAM role that Drone is permitted to assume that grants Kubernetes deployment rights. |
| EKS_CLUSTER     | required  | Name of the EKS cluster, as show in the EKS console/API.                                        |
| REPO            | required  | Docker image repository (eg `6789.dkr.ecr.us-east-1.amazonaws.com/sampleapp`)                   |
| TAG             | required  | Docker image tag (eg `latest`)                                                                  |
| NAMESPACE       | required  | Which Kubernetes namespace to operate in.                                                       |
| DEPLOYMENT      | required  | Kubernetes deployment to modify.                                                                |
| CONTAINER       | required  | Name of the container inside the deployment to update.                                          |
| AWS_REGION      | optional  | Region where the EKS cluster is located. Defaults to the same region as the Drone build agent.  |

For example, a typical deployment statement:

    deploy:
      image: drone-eks-plugin:latest
      iam_role_arn: arn:aws:iam::123456789:role/k8-deployer-role
      eks_cluster: myfirstcluster
      namespace: default
      deployment: sampleapp
      repo: 123456789.dkr.ecr.us-east-1.amazonaws.com/sampleapp
      container: sampleapp
      tag: latest
      when:
        branch: master


# Usage (in commands mode)

In some situations, you may have an application that requires some very specific
handling with `kubectl` - for example, you may need to run some queries, update
a config map or do something else that isn't the typical "deploy a new container"
type task.

To support this use case, this plugin can be executed as a regular container in
Drone with commands passed to it. The way to execute changes somewhat compared
to doing a deployment, since we need to pass the configuration parameters as
environmentals rather than arguments to the plugin.

    do-something-with-kubectl:
      image: drone-eks-plugin:latest
      pull: true
      commands:
        - /bin/connect-eks.sh
        - kubectl get pods # replace with your desired commands.
      environment:
        PLUGIN_IAM_ROLE_ARN: arn:aws:iam::123456789:role/k8-deployer-role
        PLUGIN_EKS_CLUSTER: myfirstcluster

| Environmental   | Required? | Details                                                                                         |
|-----------------|-----------|-------------------------------------------------------------------------------------------------|
| IAM_ROLE_ARN    | required  | ARN for an IAM role that Drone is permitted to assume that grants Kubernetes deployment rights. |
| EKS_CLUSTER     | required  | Name of the EKS cluster, as show in the EKS console/API.                                        |
| AWS_REGION      | optional  | Region where the EKS cluster is located. Defaults to the same region as the Drone build agent.  |

It is required that `/bin/connect-eks.sh` is the first command executed. This
runs the script that establishes the EKS connection so that `kubectl` is
correctly configured and ready to use with subsequent commands.

You will also need to adjust the `ClusterRole` that Drone uses, otherwise you
will receive plenty of `Error from server (Forbidden)`. Take extreme care when
adjusting these roles, since you may grant more permissions that you intend and
compromise the security of your Kubernetes cluster.


# Installation.

Build container with `docker build -t drone-eks-plugin:latest .` and push to
your desired registry.

In order for this plugin to work, your Drone build agents must have an IAM
Instance Profile which has the ability to assume the IAM role as defined in
`IAM_ROLE_ARN`. This role in turn must be configured to permit deployment into
Drone in the aws-auth config map.

To do this, you will need:

1. A config map for `aws-auth` that looks something like:
    ```
    mapRoles:
    ----
    ...
    - rolearn: arn:aws:iam::123456789:role/k8-deployer-role
     username: kubernetes-deployer-role
     groups:
       - deployer
    ```

2. A ClusterRole that defines the RBAC for deploying:
    ```
    rules:
    - apiGroups:
      - extensions
      resources:
      - deployments
      verbs:
      - get
      - list
      - patch
      - update
    ```

3. A binding that links that IAM ARN in `aws-auth` config map by binding the
   ClusterRole to the Group.
    ```
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: deployer
    subjects:
    - apiGroup: rbac.authorization.k8s.io
      kind: Group
      name: deployer
    ```

4. An IAM role which can be assumed by the Drone Agent Instance Profile. This
   roles does not need any policies attached, but it does need an assume role
   policy (sometimes called a "Trust Relationship"):
   ```
   {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Sid": "",
          "Effect": "Allow",
          "Principal": {
            "AWS": "arn:aws:iam::123456789:root"
          },
          "Action": "sts:AssumeRole"
        }
      ]
    }
    ```

   The Drone Agent needs a policy attached that allows it to assume the role:
   ```
   {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Sid": "",
              "Effect": "Allow",
              "Action": "sts:AssumeRole",
              "Resource": "arn:aws:iam::*:role/k8-deployer-role"
          }
      ]
    }
    ```

# Bugs, contributions, etc.

All contributions welcome - please raise a PR.
