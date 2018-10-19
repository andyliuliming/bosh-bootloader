# Patch: cfcr-azure

Steps to deploy cf with bbl:

1. Follow the normal steps to bbl up with a patch.

    ```bash
    export BOSH_BOOTLOADER=<YOUR BOSH BOOTLOADER PATH>
    mkdir banana-env && cd banana-env
    bbl plan --name banana-env
    cp -r $BOSH_BOOTLOADER/plan-patches/cfcr-azure/. .
    bbl up
    eval "$(bbl print-env)"
    ```

2. export KD as your path to kubo-deployment so you can copy-paste from below if you so desire.

    ```bash
    git clone https://github.com/cloudfoundry-incubator/kubo-deployment.git
    export KD=$(pwd)/kubo-deployment
    ```

3. upload the kubo-release

    ```bash
    git clone https://github.com/cloudfoundry-incubator/kubo-release.git
    pushd ./kubo-release
    bosh create-release && bosh upload-release
    popd
    ```

4. Upload the stemcell required
    you can build the latest xenial stemcell from this repo:
    https://github.com/cloudfoundry/bosh-linux-stemcell-builder
    and then use it for the deployment.

    or you can just use the candidate release.
    ```bash
    bosh upload-stemcell https://s3.amazonaws.com/bosh-core-stemcells-candidate/azure/bosh-stemcell-156-azure-hyperv-ubuntu-xenial-go_agent.tgz
    ```

5. Create cloud config for the deployment

    ```bash
    export deployment_name="azurecfcr"
    bosh update-config --name ${deployment_name} \
    ./ops/cfcr-vm-extensions.yml \
    --type cloud \
    -v deployment_name=${deployment_name} \
    -l <(bbl outputs)
    ```

6. Deploy the cfcr manifest.

   Notes: if you only want to do a test, use:
            -o ./ops/small-vm.yml \
          if you want to preview the manifest before apply it. use the "bosh interpolate"

    ```bash
    bosh -n -d ${deployment_name} deploy ${KD}/manifests/cfcr.yml \
    -o ${KD}/manifests/ops-files/misc/single-master.yml \
    -o ${KD}/manifests/ops-files/add-hostname-to-master-certificate.yml \
    -o ${KD}/manifests/ops-files/use-runtime-config-bosh-dns.yml \
    -o ${KD}/manifests/ops-files/rename.yml \
    -o ./ops/use-latest-kubo-release.yml \
    -o ./ops/use-vm-extensions.yml \
    -o ./ops/single-worker.yml \
    -o ./ops/use-cfcr-subnet.yml \
    -o ./ops/cloud-provider.yml \
    -v deployment_name=${deployment_name} \
    -v cfcr_location=${BBL_AZURE_REGION} \
    -l <(bbl outputs)

    bosh -d ${deployment_name} run-errand apply-specs
    ```

7. Use the deployed CFCR cluster:
    ```bash
    credhub login
    api_hostname="$(bosh int ./vars/director-vars-file.yml --path /api-hostname)"
    export DIRECTOR_NAME=<YOUR DIRECTOR NAME>
    ${KD}/bin/set_kubeconfig ${DIRECTOR_NAME}/${deployment_name} https://${api_hostname}:8443
    ```
