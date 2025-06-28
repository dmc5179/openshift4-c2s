# Install: User Provisioned Infrastructure in GovCloud (UPI) with API integration

## Create install-config.yaml

Sample install-config.yaml for govcloud can be found here:
These steps assume the cluster directory is in /home/ec2-user/cluster and the original install-config.yaml is in /home/ec2-user/install-config.yaml
- Note: GovCloud doesn't really have public hosted zone. This means that only publish==internal is supported.
- Note: Make sure to remove the "hostedZone" from the install-config.yaml if you do not intent to use Route53

## Generate manifests
```console
rm -rf cluster
mkdir cluster
cp /home/ec2-user/install-config.yaml /home/ec2-user/cluster/install-config.yaml
openshift-install-fips create manifests --dir=/home/ec2-user/cluster
```

### Get Cluster InfraID

- The cluster infra ID will be used in all of the CF templates along with the cluster name
- The cluster infra ID is in the form <clustername-XXXX>
- Run the following command to get the InfraID
```sh
jq -r '."*installconfig.ClusterID".InfraID' ./cluster/.openshift_install_state.json
```

### Remove Route53 Integration if needed
```console
cd manifests
python -c '
import yaml;
path = "manifests/cluster-dns-02-config.yml";
data = yaml.load(open(path));
del data["spec"]["privateZone"];
open(path, "w").write(yaml.dump(data, default_flow_style=False))'
```

### Generate Ignition Configs
```console
openshift-install-fips create ignition-configs --dir=/home/ec2-user/cluster
```

### Copy Ignition Configs to an S3 bucket

```console
aws s3 cp cluster/bootstrap.ign s3://mybucket/bootstrap.ign
aws s3 cp cluster/master.ign s3://mybucket/master.ign
aws s3 cp cluster/worker.ign s3://mybucket/worker.ign
```

### Deploy 02_cluster_infra.yaml Cloud Formation Template

- After deploying the cluster infra CF template, and when not using Route53 for DNS, update your DNS server such that the api.<cluster-name>.<cluster-domain> and api-int.<cluster-name>.<cluster-domain> DNS records point to the new internal AWS load balancer

### Deploy 03_cluster_security.yaml Cloud Formation Template

- Deploy the cluster security CF template

### Deploy 04_cluster_bootstrap.yaml Cloud Formation Template

- After deploying the bootstrap CF template, attach the bootstrap node to the aint and sint TargetGroups of the internal AWS load balancer.

### Deploy 05_cluster_master_nodes.yaml Cloud Formation Template

- After deploying the master nodes CF template, attach the master nodes to the aint and sint TargetGroups of the internal AWS load balancer.

### Create DNS record for ingress load balancer

- If Route53 is not being used, wait for the ingress AWS load balancer to be created by the cluster. Once the ingress AWS load balancer is active, update your DNS server such that the *.apps.<cluster-name>.<cluster-domain> DNS record points to the new ingress AWS load balancer.

### Wait for cluster install to complete

- Use the standard commands to follow the installation

```console
openshift-install-fips wait-for bootstrap-complete --dir=/home/ec2-user/cluster --log-level=debug

openshift-install-fips wait-for install-complete --dir=/home/ec2-user/cluster --log-level=debug
```