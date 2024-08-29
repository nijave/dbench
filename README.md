# dbench

This is a fork/copy from https://github.com/leeliu/dbench

Benchmark Kubernetes persistent disk volumes with `fio`: Read/write IOPS, bandwidth MB/s and latency.

# Usage

1. Download [dbench.yaml](https://gitlab.stolenleadsmen.com/infrastructure/containers/dbench/-/raw/dbench.yaml?ref_type=heads&inline=false)
   and edit the `storageClassName` to match your Kubernetes provider's Storage Class `kubectl get storageclasses`
2. Deploy Dbench using: `kubectl apply -f dbench.yaml`
3. Once deployed, the Dbench Job will:
    * provision a Persistent Volume of `1000Gi` (default) using the configured storage class or cluster default, if none were configured.
    * run a series of `fio` tests on the newly provisioned disk
    * currently there are 9 tests, 15s per test - total runtime is ~2.5 minutes
4. Follow benchmarking progress using: `kubectl logs -f job/dbench` (empty output means the Job not yet created, or `storageClassName` is invalid, see Troubleshooting below)
5. At the end of all tests, you'll see a summary that looks similar to this:
```
==================
= Dbench Summary =
==================
Random Read/Write IOPS: 75.7k/59.7k. BW: 523MiB/s / 500MiB/s
Average Latency (usec) Read/Write: 183.07/76.91
Sequential Read/Write: 536MiB/s / 512MiB/s
Mixed Random Read/Write IOPS: 43.1k/14.4k
```
1. Once the tests are finished, clean up using: `kubectl delete -f fbench.yaml` and that should deprovision the persistent disk and delete it to minimize storage billing.

```yaml
# dbench.yaml

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: dbench-pv-claim
spec:
  storageClassName: piraeus-ssd-r3  # change this to your storage class
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi

---
apiVersion: batch/v1
kind: Job
metadata:
  name: dbench
spec:
  template:
    spec:
      containers:
      - name: dbench
        image: docker.io/mmansell83/dbench:latest
        imagePullPolicy: Always
        env:
          - name: DBENCH_MOUNTPOINT
            value: /data
          # - name: DBENCH_QUICK
          #   value: "yes"
          # - name: FIO_SIZE
          #   value: 10G
          # - name: FIO_OFFSET_INCREMENT
          #   value: 256M
          # - name: FIO_DIRECT
          #   value: "0"
        volumeMounts:
        - name: dbench-pv
          mountPath: /data
      restartPolicy: Never
      volumes:
      - name: dbench-pv
        persistentVolumeClaim:
          claimName: dbench-pv-claim
  backoffLimit: 4
```

## Notes / Troubleshooting

* If the Persistent Volume Claim is stuck on Pending, it's likely you didn't specify a valid Storage Class. Double check using `kubectl get storageclasses`. Also check that the volume size of `1000Gi` (default) is available for provisioning.
* It can take some time for a Persistent Volume to be Bound and the Kubernetes Dashboard UI will show the Dbench Job as red until the volume is finished provisioning.
* It's useful to test multiple disk sizes as most cloud providers price IOPS per GB provisioned. So a `4000Gi` volume will perform better than a `1000Gi` volume. Just edit the yaml, `kubectl delete -f fbench.yaml` and run `kubectl apply -f fbench.yaml` again after deprovision/delete completes.
* A list of all `fio` tests are in [docker-entrypoint.sh](https://github.com/openebs/fbench/blob/master/docker-entrypoint.sh).

## Build Container

Instructions to build this container using podman. This will be a multi-architecture build so the normal build
process is slightly different.

The source for the below instructions is
<https://developers.redhat.com/articles/2023/11/03/how-build-multi-architecture-container-images#testing_multi_architecture_containers>

```bash
# Initialize the manifest
podman manifest create docker.io/mmansell83/dbench

# Build the image, attaching them to the manifest
podman build --platform linux/amd64,linux/arm64 --manifest docker.io/mmansell83/dbench -f Dockerfile .

# Publish the manifest
podman manifest push docker.io/mmansell83/dbench
```

Use the following to debug the built container

```bash
podman run -it --entrypoint="" localhost/dbench /bin/ash
```

## Contributors

* Lee Liu (LogDNA)
* [Alexis Turpin](https://github.com/alexis-turpin)
* [Kiran Mova](https://github.com/kmova)
* [Michael Mansell](https://github.com/mmansell83)

## License

* MIT
