# tldr

This page-tree documents the setup of a detection stack, which is fully compatible with GOAD on proxmox.
It instantiates (in its default configuration) a lightweight kubernetes stack (k3s) with three nodes (master, worker, etcd).
On this k3s deployment, I instantiate a lightweight but complete elasticsearch stack, including kibana, elasticsearch, and fleet server.
I then set up ansible roles using the generated GOAD hosts.ini to deploy the elastic agent, Sysmon and other monitoring configurations to the GOAD environment.
This allows me to turn my proxmox GOAD environment into a fully featured purple-team environment.

The following components are used to make this happen:

## Packer
Packer allows me to generate specific, preconfigured templates, which are used as base images for terraform to create the k3s cluster members.

## Terraform
Terraform is used to define the actual infrastructure deployment, and serves as a souce of truth as to what is currently running and in what configuration.
It also allows for the almost arbitrary expansion of this setup with minimal effort and re-configuration necessary.

## Ansible
Ansible is used as configuration and software management system. It allows me to configure and manage my k3s cluster, as well as deploy agents and configurations to the GOAD environment.

## Kubernetes
The k3s cluster was a deliberate choice. While running kubernetes is generally a big operational overhead, I surmised that the ability to quickly deploy new services of my chosing (should i decide to expand the lab),
and take advantage of the ECK Operator and the ease of operation regarding elasticsearch this brings, together with k3s being a lighweight and easy-to-manage distribution, makes this tradeoff worth it.
In addition, I am generally curious about kubernetes and like learning new things, as well as having some experience managing a cluster in production, make this palatable.

