# 1-click-bosh-lite-pipeline

Okay, so it's not exactly "1 click", but this repository makes it trivial to deploy a BOSH Lite VM in the Cloud and manage it using a Concourse pipeline.

The instructions here can in theory be used for any Cloud Provider, but we'll focus on the IBM Cloud, aka Softlayer, as this is where the yamls in this repository were tested on.

This guide shows 2 ways of deploying BOSH Lite into the cloud:
1. Just a BOSH Lite
2. A BOSH Lite VM in the cloud plus a Concourse management pipeline to conveniently delete, re-create, etc. that environment.

## Prerequisites:

```bash
git clone https://github.com/cloudfoundry/bosh-deployment ~/workspace/bosh-deployment
git clone https://github.com/petergtz/1-click-bosh-lite-pipeline ~/workspace/1-click-bosh-lite-pipeline
```

## Manually Creating a BOSH Lite without Generating a Concourse Management Pipeline

_Don't run this step, if you want a Concourse pipeline instead to management your BOSH Lite in the Cloud. Skip directly to [the section below](#creating-a-bosh-lite-using-a-concourse-management-pipeline) in that case._

```bash
mkdir -p ~/deployments/bosh-lite-in-sl
cd ~/deployments/bosh-lite-in-sl

sudo bosh create-env --state ./state.json \
    ~/workspace/bosh-deployment/bosh.yml \
    --vars-store=vars.yml \
    -o ~/workspace/bosh-deployment/softlayer/cpi.yml \
    -v sl_vlan_public=<PROVIDE> \
    -v sl_vlan_private=<PROVIDE> \
    -v sl_datacenter=<PROVIDE> \
    -v internal_ip=127.0.0.1 \
    -v sl_vm_domain=<PROVIDE> \
    -v sl_vm_name_prefix=<PROVIDE> \
    -v sl_username=<PROVIDE> \
    -v sl_api_key=<PROVIDE> \
    -v director_name=bosh \
    -o ~/workspace/bosh-deployment/bosh-lite.yml \
    -o ~/workspace/bosh-deployment/bosh-lite-runc.yml \
    -o ~/workspace/1-click-bosh-lite-pipeline/operations/change-to-single-dynamic-network-named-default.yml \
    -o ~/workspace/1-click-bosh-lite-pipeline/operations/change-cloud-provider-mbus-host.yml \
    -o ~/workspace/1-click-bosh-lite-pipeline/operations/make-it-work-again-workaround.yml \
    -o ~/workspace/1-click-bosh-lite-pipeline/operations/add-etc-hosts-entry.yml
```

Where the variables are defined as:
- `sl_vlan_public`, `sl_vlan_private`: The numeric IDs of the VLans as they appear in Softlayer
- `sl_datacenter`: The Softlayer datacenter, e.g. `ams03`.
- `sl_vm_name_prefix`: An arbitrary prefix for the VM name.
- `sl_vm_domain`: An arbitrary domain for the VM name. The full name of the VM will be `sl_vm_name_prefix.sl_vm_domain`
- `sl_username`,`sl_api_key`: This information can be found on your [Softlayer Profile](https://control.softlayer.com/account/user/profile) under **API Access Information** .

Now you alias the environment and set up login credentials:

```bash
bosh alias-env my-bosh -e <sl_vm_name_prefix>.<sl_vm_domain> --ca-cert <(bosh int ./vars.yml --path /director_ssl/ca)
export BOSH_CLIENT=admin
export BOSH_CLIENT_SECRET=`bosh int ./vars.yml --path /admin_password`
```

Confirm that it works:
```bash
bosh -e my-bosh env

Using environment '<sl_vm_name_prefix>.<sl_vm_domain>' as '?'

Name: ...
User: admin

Succeeded
```

_That's it! You can now use your BOSH Lite._


## Creating a BOSH Lite using a Concourse Management Pipeline

### Generating the manifest:
```bash
bosh interpolate ~/workspace/bosh-deployment/bosh.yml \
    -o ~/workspace/bosh-deployment/softlayer/cpi.yml \
    -v sl_vlan_public=<PROVIDE>
    -v sl_vlan_private=<PROVIDE>
    -v sl_datacenter=<PROVIDE>
    -v internal_ip=127.0.0.1 \
    -v sl_vm_domain=<PROVIDE> \
    -v sl_vm_name_prefix=<PROVIDE> \
    -v sl_username=<PROVIDE> \
    -v sl_api_key=<PROVIDE> \
    -v director_name=bosh \
    -o ~/workspace/bosh-deployment/bosh-lite.yml \
    -o ~/workspace/bosh-deployment/bosh-lite-runc.yml \
    -o ~/workspace/1-click-bosh-lite-pipeline/operations/change-to-single-dynamic-network-named-default.yml \
    -o ~/workspace/1-click-bosh-lite-pipeline/operations/change-cloud-provider-mbus-host.yml \
    -o ~/workspace/1-click-bosh-lite-pipeline/operations/make-it-work-again-workaround.yml \
    -o ~/workspace/1-click-bosh-lite-pipeline/operations/add-etc-hosts-entry.yml \
    > bosh-lite-in-sl.yml
```

Where the variables are defined as [above](#manually-creating-a-bosh-lite-without-generating-a-concourse-management-pipeline).

### Setting up the Configuration for the Concourse pipeline

Create a file `config.yml` with the following content:
```yaml
meta:
  bosh-lite-name: my-bosh-lite
  state-git-repo: my-private-git-repo-that-will-contain-secrets
  cf-system-domain: my.bosh-lite.system.domain.com
```

You should replace the variables with proper values:
- `bosh-lite-name`: an arbitrary name you choose. It's used for the job names in the pipeline.
- `state-git-repo`: a **private** git repository to which you have write access. It will be used to store `state.json`, the `/etc/hosts` entry created by the Softlayer CPI, and `vars.yml` that will contain the secrets. **It must not be publicly readable.**
- `cf-system-domain`: a custom CF system_domain which points to the IP that is contained in the `/etc/hosts` entry. (This can also be bosh-lite.com, but then Smoke Tests will only run as errand.)

## Creating the Concourse Pipeline to Manage the BOSH Lite VM

```bash
fly \
  -t my-target \
  set-pipeline \
  -p my-pipeline \
  -c <(spruce --concourse merge ~/workspace/1-click-bosh-lite-pipeline/template.yml ~/workspace/1-click-bosh-lite-pipeline/deploy-and-test-cf.yml config.yml) \
  -v github-private-key=<PROVIDE> \
  -v bosh-manifest="$(sed -e 's/((/_(_(/g' bosh-lite-in-sl.yml )"
```

The `sed` command in the last line is needed, because otherwise Concourse would try to interpret the `((...))` in the manifest. It's basically "escaping" the manifest. The jobs in the pipeline appropriately unescape it.

_That's it! Go to your pipeline and let it run!_
