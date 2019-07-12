---
title: What is Helm-Vault?
published: true
categories:
  - 'Blog'
tag:
  - 'Dev'
canonical_url: https://medium.com/@just_insane/what-is-helm-vault-b82949ae6ba2
---

[Helm-Vault](https://github.com/Just-Insane/Helm-Vault) is a new application designed to protect secrets contained in Helm Chart’s values.yaml files.

#### The Problem:

The problem with using Helm with Kubernetes is that there is no good way to secure your private configuration items stored in the YAML configuration files.

There are multiple reasons you may want to do this, such as to audit who is accessing what secrets, or to prevent unauthorized modifications to the environment. Another reason may be that you want to have publicly available documentation without worrying about scrubbing the files.

#### Current Solutions:

Currently available solutions require you to significantly modify the Helm Chart, or encrypt the entire document with GPG or a hosted KMS solution.

This can be a pain to manage, who wants to use GPG all the time to work with files? What happens if the key is lost? What happens if the internet is down and you can’t access your KMS provider?

#### What makes Helm-Vault different:

With Helm-Vault, you get full access to the structure and content of the YAML documents, even when they are in an “encrypted” state, this provides you a lot more flexibility. When publishing charts, you can seamlessly provide your encrypted YAML files, and with the deliminator, provide easily notable locations where information needs to be changed.

If you run Hashicorp [Vault](https://www.vaultproject.io/) internally, you will always have access to your secret data, and don’t have to worry about working with GPG keys, which makes life a lot easier for your developers.

#### Getting Started:

As long as you have Python 3.7, a working Hashicorp Vault environment, and a Vault token, you can get started with Helm-Vault in 2 easy steps:

1. `pip3 install git+https://github.com/Just-Insane/helm-vault`
2. `helm plugin install https://github.com/Just-Insane/helm-vault`

You can find information about using Helm-Vault in the [README](https://github.com/Just-Insane/helm-vault/blob/master/README.md#usage-and-examples).

If you have any questions or concerns, feel free to open an issue on the project’s [GitHub Issue Tracker](https://github.com/Just-Insane/helm-vault/issues).