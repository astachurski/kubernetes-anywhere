# Deploying a Kubernetes Development Cluster

This describes the steps to use kubernetes-anywhere to deploy a custom build of
kubernetes in GCE on a Linux distro that has the apt package manager.

- gcp-project-id = the Google Cloud Platform project id where you want your dev
  cluster to run.
- gcs-bucket-name = GCS bucket you will create to hold some artifacts needed
  during the node installation process. Normally this will be
  "gcr.io/&lt;gcp-project-id&gt;"

1. Create a GCS bucket to hold your custom Debian packages for kubeadm and the
   other components used to bootstrap the cluster. Make it public so that during
   the installation process, when the `phase1/gce/configure-vm-kubeadm.sh`
   script runs on the destination VMs, it will be able to access the bucket and
   download the Debian packages it needs.

    ```sh
    GCS_BUCKET_NAME="<gcs-bucket-name>"
    gsutil mb gs://${GCS_BUCKET_NAME}
    gsutil defacl ch -u AllUsers:R gs://${GCS_BUCKET_NAME}
    ```

2. Create a `.config` file in the root directory of the kubernetes-anywhere
   project.

    ```
    #
    # Phase 1: Cluster Resource Provisioning
    #
    .phase1.num_nodes=2
    .phase1.cluster_name="k-a"
    .phase1.cloud_provider="gce"

    #
    # GCE configuration
    #
    .phase1.gce.os_image="ubuntu-1604-xenial-v20160420c"
    .phase1.gce.instance_type="n1-standard-2"
    .phase1.gce.project="<gcp-project-id>"
    .phase1.gce.region="us-central1"
    .phase1.gce.zone="us-central1-b"
    .phase1.gce.network="default"

    #
    # Phase 2: Node Bootstrapping
    #
    .phase2.installer_container="docker.io/colemickens/k8s-ignition:latest"
    .phase2.docker_registry="gcr.io/<gcp-project-id>"
    .phase2.kubernetes_version="v1.6.0-alpha"
    .phase2.provider="kubeadm"
    .phase2.kubeadm.version="gs://<gcs-bucket-name>/build/debs"

    #
    # Phase 3: Deploying Addons
    #
    .phase3.run_addons=y
    .phase3.kube_proxy=y
    .phase3.dashboard=y
    .phase3.heapster=y
    .phase3.kube_dns=y

    ## Setup Steps
    ```

3. Set the following values:

    * .phase1.gce.project
    * .phase2.docker_registry
    * .phase2.kubernetes_version - this should match the version information
      used in the GCR tag for the images you want to run. Specifically it
      should match the value in the upload.sh script down below.
    * .phase2.kubeadm.version="gs://&lt;gcs-bucket-name&gt;/build/debs"

4. Modify `phase1/gce/configure-vm-kubeadm.sh` to include this line:

    ```diff
     KUBERNETES_VERSION=$(get_metadata "k8s-kubernetes-version")
    +export KUBE_REPO_PREFIX="gcr.io/<gcp-project-id>"
    ```

5. There are some additional images used during the kubernetes install that are
   not built as part of kubernetes. Tag recent versions of these images with the
   same GCR as your are using for your build.

   Replace gcp-project-id with your project id.

    ```sh
    GCP_PROJECT_ID=<gcp-project-id>
    gcloud docker -a

    docker pull gcr.io/google_containers/etcd-amd64:3.0.14-kubeadm
    docker tag gcr.io/google_containers/etcd-amd64:3.0.14-kubeadm gcr.io/${GCP_PROJECT_ID}/etcd-amd64:3.0.14-kubeadm
    docker push gcr.io/${GCP_PROJECT_ID}/etcd-amd64:3.0.14-kubeadm

    docker pull gcr.io/google-containers/pause-amd64:3.0
    docker tag gcr.io/google-containers/pause-amd64:3.0 gcr.io/${GCP_PROJECT_ID}/pause-amd64:3.0
    docker push gcr.io/${GCP_PROJECT_ID}/pause-amd64:3.0

    docker pull gcr.io/google-containers/kube-discovery-amd64:1.0
    docker tag gcr.io/google-containers/kube-discovery-amd64:1.0 gcr.io/${GCP_PROJECT_ID}/kube-discovery-amd64:1.0
    docker push gcr.io/${GCP_PROJECT_ID}/kube-discovery-amd64:1.0
    ```

6. Create a shell script like this in your kubernetes directory, called
   `upload.sh`. There is an improvement for this step in flight.

    ```sh
    #!/bin/bash

    set -o xtrace
    set -o nounset
    set -o pipefail
    set -o errexit

    # The version to use for tagging the image in GCR. This value should
    # correspond to the value set for .phase2.kubernetes_version in the .config
    # file above.
    declare -r KUBE_VERSION="v1.6.0-alpha"

    if [[ -z ${KUBE_REPO_PREFIX:-} ]]; then
      echo "You must define \$KUBE_REPO_PREFIX. Something like:"
      echo
      echo "export KUBE_REPO_PREFIX=gcr.io/<gcp_project_name>"
      echo
      exit 1
    fi

    build_and_push() {
      local component="$1"
      bazel run "//build:${component}"
      docker tag "gcr.io/google-containers/build:${component}" "${KUBE_REPO_PREFIX}/${component}-amd64:${KUBE_VERSION}"
      docker push "${KUBE_REPO_PREFIX}/${component}-amd64:${KUBE_VERSION}"
    }

    components=("${@}")

    for component in "${components[@]}"; do
      build_and_push "${component}"
    done
    ```

7. Upload a custom builds of the listed components to GCR.

    ```sh
    cd kubernetes
    gcloud docker -a
    bash upload.sh kube-apiserver kube-controller-manager kube-proxy kube-scheduler
    ```

8. Run these commands to create a cluster.

    ```sh
    make docker-dev
    make deploy-cluster
    ```

9. When you are done you can destroy the cluster.

    ```sh
    make destroy-cluster
    ```

## Refreshing the Images in an Existing Cluster

Once you have a cluster running, you may find it convenient to redeploy changes
to one or more components without completely recreating the cluster. These steps
describe a process for accomplishing that.

1. Upload new image(s) to GCR. You can specify one or more images as parameters
   to the `upload.sh` script, depending on what you are making changes to. In
   this example, only kube-apiserver is specified.

    ```sh
    cd kubernetes
    gcloud docker -a
    bash upload.sh kube-apiserver
    ```

2. Use the new image in the cluster by removing the old image on the master,
   then removing the container and allowing it to be automatically restarted.
   During the restart, the image will not be found locally, so the new image
   will be downloaded from GCR.

    ```sh
    sudo docker rmi -f gcr.io/google_containers/kube-apiserver-amd64:v1.5.1
    sudo docker rm -f `sudo docker ps --filter name=kube-apiserver -q`
    sudo docker ps --filter name=kube-apiserver
    ```

