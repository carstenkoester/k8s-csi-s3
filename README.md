# CSI for S3

This is a Container Storage Interface ([CSI](https://github.com/container-storage-interface/spec/blob/master/spec.md)) for S3 (or S3 compatible) storage. This can dynamically allocate buckets and mount them via a fuse mount into any container.

## Kubernetes installation

### Requirements

* Kubernetes 1.17+
* Kubernetes has to allow privileged containers
* Docker daemon must allow shared mounts (systemd flag `MountFlags=shared`)

### 1. Create a secret with your S3 credentials

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: csi-s3-secret
  # Namespace depends on the configuration in the storageclass.yaml
  namespace: kube-system
stringData:
  accessKeyID: <YOUR_ACCESS_KEY_ID>
  secretAccessKey: <YOUR_SECRET_ACCESS_KEY>
  # For AWS set it to "https://s3.<region>.amazonaws.com", for example https://s3.eu-central-1.amazonaws.com
  endpoint: https://storage.yandexcloud.net
  # For AWS set it to AWS region
  #region: ""
```

The region can be empty if you are using some other S3 compatible storage.

### 2. Deploy the driver

```bash
cd deploy/kubernetes
kubectl create -f provisioner.yaml
kubectl create -f driver.yaml
kubectl create -f csi-s3.yaml
```

If you're upgrading from a previous version which had `attacher.yaml` you
can safely delete all resources created from that file:

```
wget https://raw.githubusercontent.com/yandex-cloud/k8s-csi-s3/v0.35.5/deploy/kubernetes/attacher.yaml
kubectl delete -f attacher.yaml
```

### 3. Create the storage class

```bash
kubectl create -f examples/storageclass.yaml
```

### 4. Test the S3 driver

1. Create a pvc using the new storage class:

    ```bash
    kubectl create -f examples/pvc.yaml
    ```

1. Check if the PVC has been bound:

    ```bash
    $ kubectl get pvc csi-s3-pvc
    NAME         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    csi-s3-pvc   Bound     pvc-c5d4634f-8507-11e8-9f33-0e243832354b   5Gi        RWO            csi-s3         9s
    ```

1. Create a test pod which mounts your volume:

    ```bash
    kubectl create -f examples/pod.yaml
    ```

    If the pod can start, everything should be working.

1. Test the mount

    ```bash
    $ kubectl exec -ti csi-s3-test-nginx bash
    $ mount | grep fuse
    s3fs on /var/lib/www/html type fuse.s3fs (rw,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other)
    $ touch /var/lib/www/html/hello_world
    ```

If something does not work as expected, check the troubleshooting section below.

## Additional configuration

### Bucket

By default, csi-s3 will create a new bucket per volume. The bucket name will match that of the volume ID. If you want your volumes to live in a precreated bucket, you can simply specify the bucket in the storage class parameters:

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: csi-s3-existing-bucket
provisioner: ru.yandex.s3.csi
parameters:
  mounter: geesefs
  options: "--memory-limit 1000 --dir-mode 0777 --file-mode 0666"
  bucket: some-existing-bucket-name
```

If the bucket is specified, it will still be created if it does not exist on the backend. Every volume will get its own prefix within the bucket which matches the volume ID. When deleting a volume, also just the prefix will be deleted.

### Static Provisioning

If you want to mount a pre-existing bucket or prefix within a pre-existing bucket and don't want csi-s3 to delete it when PV is deleted, you can use static provisioning.

To do that you should omit `storageClassName` in the `PersistentVolumeClaim` and manually create a `PersistentVolume` with a matching `claimRef`, like in the following example: [deploy/kubernetes/examples/pvc-manual.yaml](deploy/kubernetes/examples/pvc-manual.yaml).

### Mounter

We **strongly recommend** to use the default mounter which is [GeeseFS](https://github.com/yandex-cloud/geesefs).

However there is also support for two other backends: [s3fs](https://github.com/s3fs-fuse/s3fs-fuse) and [rclone](https://rclone.org/commands/rclone_mount).

The mounter can be set as a parameter in the storage class. You can also create multiple storage classes for each mounter if you like.

As S3 is not a real file system there are some limitations to consider here.
Depending on what mounter you are using, you will have different levels of POSIX compability.
Also depending on what S3 storage backend you are using there are not always [consistency guarantees](https://github.com/gaul/are-we-consistent-yet#observed-consistency).

You can check POSIX compatibility matrix here: https://github.com/yandex-cloud/geesefs#posix-compatibility-matrix.

#### GeeseFS

* Almost full POSIX compatibility
* Good performance for both small and big files
* Does not store file permissions and custom modification times
* By default runs **outside** of the csi-s3 container using systemd, to not crash
  mountpoints with "Transport endpoint is not connected" when csi-s3 is upgraded
  or restarted. Add `--no-systemd` to `parameters.options` of the `StorageClass`
  to disable this behaviour.

#### s3fs

* Almost full POSIX compatibility
* Good performance for big files, poor performance for small files
* Very slow for directories with a large number of files

#### rclone

* Poor POSIX compatibility
* Bad performance for big files, okayish performance for small files
* Doesn't create directory objects like s3fs or GeeseFS
* May hang :-)

## Advanced configuration

### IAM roles with OIDC

As an alternative to use static, long-lived credentials to access the S3 bucket, you can use a Kubernetes ServiceAccount token and OIDC integration to assume an IAM role. **At the time of writing, this has only been tested with AWS as an identity provider**. This requires configuration on the Kubernetes cluster as well as the target AWS account.

#### Prerequisites

Configuring OIDC integration between your Kubernetes cluster and the AWS account is outside of the scope of this document. For EKS clusters, there is published documentation at <https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html>. For other clusters (including non-EKS clusters in AWS, clusters in other clouds, and clusters operated on-prem), the following external links might provide helpful guidance:

  - <https://reece.tech/posts/oidc-k8s-to-aws/>
  - <https://medium.com/@channyein/kubernetes-self-managed-cluster-with-aws-iam-oidc-e483d4dc0df2>
  - <https://hangarau.space/providing-access-to-aws-resources-from-kubernetes-pods-using-oidc/>

You should have OIDC integration between your Kubernetes cluster and AWS completed before proceeding. The following files included in this repository can be used to verify the integration:

 1. If your CSI ServiceAccount does not already exist, apply the file `deploy/kubernetes/examples/oidc-test/serviceaccount.yaml`:

    ```
    $ kubectl apply -f deploy/kubernetes/examples/oidc-test/serviceaccount.yaml
    ```

    Skip this step if your ServiceAccount already exists

 2. Edit `deploy/kubernetes/examples/oidc-test/pod.yaml` and replace `__INSERT_YOUR_OIDC_AUDIENCE_HERE__` with your OIDC audience, and replace `__INSERT_YOUR_IAM_ROLE_ARN_HERE__` with your IAM role ARN. **If you're using Amazon EKS with pre-configured IAM-Roles-for-Serviceaccounts configuration, then the audience is `sts.amazonaws.com`**. Then apply this pod:

    ```
    $ kubectl apply -f deploy/kubernetes/examples/oidc-test/pod.yaml
    ```

 3. Execute a shell inside that pod. Within that shell, you should be able to access your S3 buckets:

    ```
    $ kubectl exec -it -n kube-system test-oidc -- bash
    bash-4.2# aws s3 ls
    2020-01-01 00:00:00 bucket1
    2021-01-01 00:00:00 bucket2
    bash-4.2# exit
    ```

 4. Resolve any errors or issues before proceeding.

 5. Remove the test pod, and the serviceaccount if it was created in step 1.

#### Usage

When OIDC integration between your Kubernetes cluster and the AWS account is known to be working, then you can utilize the integration as follows:

 - When deploying the CSI plugin via the Helm chart:
     - Set `oidc.audience` to the OIDC audience you're accepting in the AWS OIDC integration
     - set `secret.iamRoleArn` to your IAM role ARN
     - Leave the `secret.accessKey` and `secret.secretKey` values blank
 - When not using the Helm chart, you'd currently need to modify `deploy/kubernetes/csi-s3.yaml` and `deploy/kubernetes/provisioner.yaml` and add [Kubernetes ServiceAccount token projection](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#launch-a-pod-using-service-account-token-projection) to the manifest. Set the `AWS_WEB_IDENTITY_TOKEN_FILE` environment variable to the path where the projected token is mounted.
 - In any Kubernetes Secret you're creating with S3 credentials, leave `accessKeyID` and `secretAccessKey` empty, and instead, include an `iamRoleArn` value with the ARN of your IAM role. If you use different secrets (for example, when doing Static Provisioning of buckets in different AWS accounts), your secrets can point to different IAM roles as needed.

## Troubleshooting

### Issues while creating PVC

Check the logs of the provisioner:

```bash
kubectl logs -l app=csi-provisioner-s3 -c csi-s3
```

### Issues creating containers

1. Ensure feature gate `MountPropagation` is not set to `false`
2. Check the logs of the s3-driver:

```bash
kubectl logs -l app=csi-s3 -c csi-s3
```

## Development

This project can be built like any other go application.

```bash
go get -u github.com/yandex-cloud/k8s-csi-s3
```

### Build executable

```bash
make build
```

### Tests

Currently the driver is tested by the [CSI Sanity Tester](https://github.com/kubernetes-csi/csi-test/tree/master/pkg/sanity). As end-to-end tests require S3 storage and a mounter like s3fs, this is best done in a docker container. A Dockerfile and the test script are in the `test` directory. The easiest way to run the tests is to just use the make command:

```bash
make test
```
