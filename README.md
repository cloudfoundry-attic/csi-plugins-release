# csi-plugins-release
This bosh release provides example packaging for Container Storage Interface (CSI) plugins that normally run as 
containers.  Specifically, it packages a kubernetes NFS CSI plugin, running in a Docker container, and a generic service 
broker that is configured to produce CSI service bindings targeted to the NFS plugin.
 
To run the CSI plugin on the Diego Cell, we create an OCI container spec file and start the container using `runc`.  See
[the CF CSI Support docs](CSI_SUPPORT.md) for details.

# Deploying to Cloud Foundry

## Pre-requisites

1. Note that this release **does not work with BOSH Lite or CF Dev**.

1. Install Cloud Foundry, or start from an existing CF deployment.  If you are starting from scratch, the article [Overview of Deploying Cloud Foundry](https://docs.cloudfoundry.org/deploying/index.html) provides detailed instructions.

## Deploy csi-local-volume-release using cf-deployment

3. Deploy csi-local-volume-release with cf

```bash
cd ~/workspace/cf-deployment
    $ bosh -e my-env -d cf deploy cf.yml -v deployment-vars.yml -o ../csi-plugins-release/operations/add-csi-nfs-plugin.yml
```
   **Note:** the above command is an example, but your deployment command should match the one you used to deploy Cloud Foundry initially, with the addition of a `-o operations/enable-nfs-volume-service.yml` option.

## Register csibroker

If not using credhub for your bosh director, you can find generated passwords in `deployment-vars.yml`
```bash
cd ~/workspace/cf-deployment
cf_password=`cat deployment-vars.yml |grep cf_admin_password|awk '{print $2}'`
broker_password=`cat deployment-vars.yml |grep csi-localbroker-password|awk '{print $2}'`
```

If using credhub:
```bash
    bosh_manifest_password_variable_name=cf_admin_password
    cs_password=`credhub find -j -n ${bosh_manifest_password_variable_name} | jq -r .credentials[].name | xargs credhub get -j -n | jq -r .value`

    bosh_manifest_password_variable_name=csi-nfsbroker-password
    broker_password=`credhub find -j -n ${bosh_manifest_password_variable_name} | jq -r .credentials[].name | xargs credhub get -j -n | jq -r .value`
```
Now register the broker:
```bash
# login with cf
cf api api.your-dowmain.com --skip-ssl-validation
cf auth admin ${cf_password}

# optionally delete previous broker:
cf delete-service-broker csilocalfs-broker

# create-service-broker and enable access
cf create-service-broker csilocalfs-broker csi-localbroker ${broker_password} http://csi-nfsbroker.your-domain.com
cf enable-service-access csi-nfs -p free
```

## Deploy pora and test volume services

```bash
cd ~/workspace/csi-local-volume-release
pushd ./src/code.cloudfoundry.org/persi-acceptance-tests/

# create a service
cf create-org test_org
cf target -o test_org

cf create-space test_space
cf target -s test_space

cf create-service csi-nfs free pora-volume-instance \
-c '{"name":"csi-nfs-storage","volume_capabilities":[{"mount":{}}],"parameters":{"server":"nfstestserver.service.cf.internal","share":"/export/users2000"}}'

# push pora and bind service
cf push pora -f ./assets/pora/manifest.yml -p ./assets/pora/ --no-start
cf bind-service pora pora-volume-instance
cf start pora
popd
```
