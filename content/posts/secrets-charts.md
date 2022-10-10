+++ 
author = "Ernesto González" 
title = "Using Sealed Secrets to encrypt secrets in a Helm Chart values file" 
date = "2022-09-16" 
description = "Using Sealed Secrets to deploy Helm Charts from GitHub" 
tags = [ "Git", "Secret Management", "GitOps", "Helm"] 
+++

Reading the title of this post you may ask yourself: "Why would I want to encrypt a secret and put it on a values file?". And to that, I would respond: "Well, to push it to GitHub (or any equivalent), of course". And you would ask me again: "But why would I want to push a secret to a public (or private, even) repository?", and I will answer: "Well! Because GitOps!".

So what is GitOps? And why is this even a problem in the first place?

## GitOps

[Red Hat](https://www.redhat.com/en/topics/devops/what-is-gitops#what-is-gitops) defines GitOps as a methodology that "uses Git repositories as a single source of truth to deliver infrastructure as code. Submitted code checks the CI process, while the CD process checks and applies requirements for things like security, infrastructure as code, or any other boundaries set for the application framework. All code changes are tracked, making updates easy while also providing version control should a rollback be needed."

The most important part of that definition is the one that says that the Git repository is the **single source of truth**. So everything related to the application you want to develop and deploy needs to be in that Git repository. And you will ask: "Does that include secrets?", and I will answer: "Maybe". There are two ways: 

1. You can store the secrets encrypted in the Git repo.
2. You can create references to the secrets and with some automation create the Kubernetes Secret.

For this article, we are going to cover the first methodology. And to be more specific, we are going to cover how to encrypt a secret with Sealed Secrets and use it in a Helm Chart.

## Helm Charts and Sealed Secrets

If you are not familiar with Helm Charts, I invite you to read more [here](https://helm.sh/docs/topics/charts/). 

And to know more about Sealed Secrets, follow this [link](https://github.com/bitnami-labs/sealed-secrets).

Usually, the way to handle sensitive information with Sealed Secrets is to create a Kubernetes `Secret` and encrypt it using `kubeseal` so it creates a `SealedSecret`. Then, you have to create that `SealedSecret` in the OpenShift cluster and the Sealed Secrets controller will decrypt it and create a K8s `Secret` with the information from the `SealedSecret`. A more detailed procedure can be found [here](https://github.com/bitnami-labs/sealed-secrets#usage).

But the idea of this blog is to give the option of encrypting the secret directly in a Helm `Values` file. So we can't create the Sealed Secret that way, we will need to use the `raw` mode and paste the secret directly into the file. For that we will need to do the following:

> **NOTE:** For this blog we are going to use the Hello Secret app, which is available at: https://github.com/ernesgonzalez33/hello-secret

### Install Sealed Secrets in OpenShift

First of all, add the Sealed Secrets repo:

```
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
```

Create the project where you want to install it:

```
oc new-project sealed-secrets
```

Install it by using the values file in this repo:

```
helm install sealed-secrets -f sealed-secrets/values.yaml sealed-secrets/sealed-secrets
```

### Clone Hello Secret repo

Clone the Hello Secret App repo:

```bash
git clone git@github.com:ernesgonzalez33/hello-secret.git
```

### Encrypt your secret

First, you will need to install kubeseal on your machine following these instructions:

* [Homebrew](https://github.com/bitnami-labs/sealed-secrets#homebrew)
* [MacPorts](https://github.com/bitnami-labs/sealed-secrets#macports)
* [Nixpkgs](https://github.com/bitnami-labs/sealed-secrets#nixpkgs)
* [Linux](https://github.com/bitnami-labs/sealed-secrets#linux)
* [From Source](https://github.com/bitnami-labs/sealed-secrets#installation-from-source)

Then, you will need to encrypt your secrets, and I'm not meaning the entire K8s secret, but only the words/passwords you need encrypted. For that, run the following command:

```bash
echo -n <my-secret> | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --raw --from-file=/dev/stdin --scope=namespace-wide
```

> **NOTE:** The scope in the command needs to match with the selected in the Values file. By default, it is `namespace`. For more info: https://github.com/bitnami-labs/sealed-secrets#scopes

The output of that command will be your secret encrypted. You will only need to paste it into the values file of the Hello Secret chart. Located under `chart/hello-secret/values.yaml`

### Deploy Hello Secret using the Helm Chart

Run the following command:

```bash
helm install hello-secret ./chart/hello-secret/
```

## Conclusion

Secrets management in GitOps is not an easy task, and every way of doing it has some advantages and disadvantages. With this blog, we only wanted to teach how to do it using Helm Charts and letting your installers encrypt the secrets directly in the `Values` file. It might not be the most secure way, but it is one of the most GitOps-y ones.

For more in-depth information about Secrets Management with GitOps, I highly recommend the following blog [post](https://cloud.redhat.com/blog/a-guide-to-secrets-management-with-gitops-and-kubernetes).