# Exercise 5

In this exercise we will start a Debian VM which acts as the [Matchbox](https://coreos.com/matchbox/docs/latest) and [DHCP server](https://www.isc.org/downloads/dhcp). All the configuration of Matchbox and the other components is done in a static manner. The provisioning scripts for the Matchbox VM can be found in `./scripts`. Like in exercise 1.

Additionally we will start an [etcd](https://coreos.com/etcd/) "cluster" on the Matchbox and we will use [confd](https://github.com/kelseyhightower/confd) to dynamically configure our DHCP server. Also we are starting [Vault](https://www.vaultproject.io) as our PKI and secret store.

The configuration for our machines is stored in two yaml files under `./data` normally these information would be stored in your asset management tool.

## Starting Matchbox

```bash
make pxe_server
```

This will take a while until all components are successfully installed. In the mean time take a look at the provisioning script and the `Makefile`.

## Render Matchbox configuration

```bash
make terraform_config
```

This will copy all files from `./terraform` onto the Matchbox VM (into the folder `/etc/terraform`). In the next step the terraform configuration will be applied against the Matchbox. This step actually renders all the ignition templates and sets the groups according to the Matchbox selectors (we will get back to selectors in a later step). In the end we run two `curl` commands to validate that the configuration was applied successfully (since terraform can't tell us all errors e.g. a missing or bad configured ignition template). The final configuration for terraform will be generated by confd on the Matchbox with the `confd-terraform.service`.

## Deploy the new hosts

```bash
make cluster
```

This command will spin up two "blank" VM's that PXE boot from the Matchbox. Currently there is an issue with PXE boot, Vagrant and the VirtualBox Provider (because of this we need to set an Vagrant box image even we don't use it at all).

SSH into the `pxe_server` with `vagrant ssh pxe_server` and look at the following services:

```bash
sudo journalctl -fu confd-dhcp
sudo journalctl -fu isc-dhcp-server
sudo journalctl -fu confd-terraform
sudo docker exec -ti etcd etcdctl -u root:rootpw ls --recursive
```

## Inspect the Matchbox

If you run `sudo docker exec -ti etcd etcdctl -u root:rootpw watch /nodes/kmaster-fluffy-unicorn-az01-001/deploy` on the Matchbox VM you can also see when the newly booted machines are reporting their "status".

## SSH into the new hosts

In order to ssh into the newly provisioned hosts you need to ssh into the `pxe_server` with `vagrant ssh pxe_server` now you can ssh into the other machines:

```bash
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i insecure_private_key core@192.168.1.2
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i insecure_private_key core@192.168.1.100
```

## Install Gitlab

You can find the complete instructions for installing gitlab [on Debian](https://about.gitlab.com/installation/#debian) and installing a [Gitlab runner](https://docs.gitlab.com/runner/install/linux-repository.html) on the Matchbox (as `Docker executor`)

## Write a pipeline

Create a new Project in Gitlab and push all the code into it. Now write a [gitlab-ci.yaml](https://docs.gitlab.com/ee/ci/yaml/) and write your first job. In the next step add some [stages](https://docs.gitlab.com/ee/ci/yaml/#stages) and rewrite all these steps from the Makefile into Gitlab CI jobs:

- `download_assets`
- `terraform_config`
- `etcd_data`
- `confd_config`
- Add an additional step `redeploy cluster` and `update cluster`

Additionally you can also deploy `kube-proxy` and `calico` as part of a Gitlab Job. Also feel free to add any other job / Kubernetes deployment that you think could be useful.

## Clean up

In order to clean everything up: `make clean`
