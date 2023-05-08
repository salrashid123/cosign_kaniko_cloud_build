# Deterministic container hashes and container signing using Cosign, Kaniko and Google Cloud Build

A simple tutorial that generates consistent container image hashes using `kankko` and then signs provenance records using [cosign](https://github.com/sigstore/cosign) (Container Signing, Verification and Storage in an OCI registry).


this is identical to [Deterministic container hashes and container signing using Cosign, Bazel and Google Cloud Build](https://github.com/salrashid123/cosign_bazel_cloud_build) except ofcourse it uses [kaniko](https://github.com/GoogleContainerTools/kaniko) and its flag that enabled reproduceability:


- [Flag reproducible](https://github.com/GoogleContainerTools/kaniko#flag---reproducible)

```
--reproducible

Set this flag to strip timestamps out of the built image and make it reproducible.
```

This will create a consistent image hash:
 
* `securebuild-kaniko@sha256:74110f406e1ad3d56301a9bb68f93118f0c8dc3170a5e114a96f7db76ad30225`

In this tutorial, we will:

1. generate a deterministic container image hash using  `kaniko`
2. use `cosign` to create provenance records for this image
3. use `syft` to generate the container `sbom`
4. use cosign to sign the container sbom
5. verify attestations and signatures using `KMS` and `OIDC` cross checked with a public transparency log.
6. use `syft` to generate the application `sbom`
7. sign the application `sbom` with cosign

I'm skipping explaining most of the steps that show usage of `cosign` and `syft`.  For those please see the `bazel` examples


We will use GCP-centric services here such as `Artifact Registry`, `Cloud BUild`, `Cloud Source Repository`.  

Both `KMS` and `OIDC` based signatures are used and for `OIDC`, an entry is submitted to a `transparency log` such that it can get verified by anyone at anytime.

>> **NOTE** Please be aware that if you run this tutorial, the GCP service_accounts _email_ you use to sign the artifacts within cloud build will be submitted to a public transparency log.  I used a disposable GCP project but even if i didn't, its just the email address and projectID in the cert, no big deal to me.  If it is to you, you can use the KMS examples and skip OIDC

>> this repo is not supported by google and employs as much as i know about it on 9/24/22 (with one weeks' experience with this..so take it with a grain of salt)

---

##### References:

* [SigStore](https://docs.sigstore.dev/)
* [cosign](https://github.com/sigstore/cosign)
* [Introducing sigstore: Easy Code Signing & Verification for Supply Chain Integrity](https://security.googleblog.com/2021/03/introducing-sigstore-easy-code-signing.html)
* [Best Practices for Supply Chain Security](https://dlorenc.medium.com/policy-and-attestations-89650fd6f4fa)
* [Deterministic builds with go + bazel + grpc + docker](https://github.com/salrashid123/go-grpc-bazel-docker)
* [in-toto attestation](https://docs.sigstore.dev/cosign/attestation/)
* [Notary V2 and Cosign](https://dlorenc.medium.com/notary-v2-and-cosign-b816658f044d)

---


### Setup

The following steps will use Google Cloud services 

* Cloud Source Repository to hold the code and trigger builds (you can use github but thats out of scope here),
* Cloud Build to create the image to save to artifact registry.
* Artifact Registry to hold the containers images

You'll also need to install [cosign](https://docs.sigstore.dev/cosign/installation/) (duh), and [rekor-cli](https://docs.sigstore.dev/rekor/installation), `git`, `gcloud`, optionally `gcloud`, `docker`.

```bash
export GCLOUD_USER=`gcloud config get-value core/account`
export PROJECT_ID=`gcloud config get-value core/project`
export PROJECT_NUMBER=`gcloud projects describe $PROJECT_ID --format='value(projectNumber)'`
echo $PROJECT_ID

## the projectID i used for this demo is PROJECT_ID=cosign-test-kaniko-1-384813

gcloud auth application-default login

# enable services
gcloud services enable \
    artifactregistry.googleapis.com \
    cloudbuild.googleapis.com cloudkms.googleapis.com \
    iam.googleapis.com sourcerepo.googleapis.com

# create artifact registry
gcloud artifacts repositories create repo1 --repository-format=docker --location=us-central1

# create service account that cloud build will run as.
gcloud iam service-accounts create cosign-kaniko

# allow 'self impersonation' for cloud build service account
gcloud iam service-accounts add-iam-policy-binding cosign-kaniko@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/iam.serviceAccountTokenCreator \
    --member "serviceAccount:cosign-kaniko@$PROJECT_ID.iam.gserviceaccount.com"

# allow cloud build to write logs
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:cosign-kaniko@$PROJECT_ID.iam.gserviceaccount.com  \
  --role=roles/logging.logWriter

# allow cloud build write access to artifact registry
gcloud artifacts repositories add-iam-policy-binding repo1 \
    --location=us-central1  \
    --member=serviceAccount:cosign-kaniko@$PROJECT_ID.iam.gserviceaccount.com \
    --role=roles/artifactregistry.writer

# allow cloud build access to list KMS keys
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:cosign-kaniko@$PROJECT_ID.iam.gserviceaccount.com  \
  --role=roles/cloudkms.viewer


# create kms keyring and key
gcloud kms keyrings create cosignkr --location=global

gcloud kms keys create key1 --keyring=cosignkr \
 --location=global --purpose=asymmetric-signing \
 --default-algorithm=ec-sign-p256-sha256

gcloud kms keys list  --keyring=cosignkr --location=global

# allow cloud buildaccess to sign the key
gcloud kms keys add-iam-policy-binding key1 \
    --keyring=cosignkr --location=global \
    --member=serviceAccount:cosign-kaniko@$PROJECT_ID.iam.gserviceaccount.com \
    --role=roles/cloudkms.signer

# allow current gcloud user to view the public key
gcloud kms keys add-iam-policy-binding key1 \
    --keyring=cosignkr --location=global \
    --member=serviceAccount:cosign-kaniko@$PROJECT_ID.iam.gserviceaccount.com  \
    --role=roles/cloudkms.publicKeyViewer

# create a temp bucket for cloud build and allow cloud build permissions to use it
gsutil mb gs://$PROJECT_ID\_cloudbuild
gsutil iam ch serviceAccount:cosign-kaniko@$PROJECT_ID.iam.gserviceaccount.com:objectAdmin gs://$PROJECT_ID\_cloudbuild
```

### Dockerfile and go VCS

The final image uses [distroless base-debian11 nonroot](https://github.com/GoogleContainerTools/distroless#what-images-are-available).  

This specific image type runs as the name suggest, as a non-root user/group (specifically `nonroot:nonroot, 65532:65532` with home directory of `/home/nonroot`)

The Dockerfile build for the image will copy over the required files and dependencies so that nonroot can invoke and access them

```dockerfile
# base-debian11-nonroot
FROM gcr.io/distroless/base-debian11@sha256:10985f0577b62767a2f0218bff7dec33c59a5ef9a587d889bf0861e0c3123d57
```

The build also disable the version control information [-buildvcs=false](https://tip.golang.org/doc/go1.18)

The only reason this was done was to show the various build options outside of VCS that arrives at the same hash.  If you used cloud build+cloud source repository commits to build, the git control information would get embedded into the final binary (which would ofcourse change the relative hash).


```dockerfile
RUN GOOS=linux GOARCH=amd64 go build -buildvcs=false -o /go/bin/server
```

### Build image

```bash
## for local docker and kaniko.

# see appendix to setup credentials for artifact registry
# cd /app
# docker run    -v `pwd`:/workspace   -v $HOME/.docker/config_kaniko.json:/kaniko/.docker/config.json:ro  \
#              gcr.io/kaniko-project/executor@sha256:034f15e6fe235490e64a4173d02d0a41f61382450c314fffed9b8ca96dff66b2    \
#               --dockerfile=Dockerfile --reproducible \
#               --destination "us-central1-docker.pkg.dev/$PROJECT_ID/repo1/securebuild-kaniko:server"     --context dir:///workspace/

## for cloud build
# cd /app
# gcloud beta builds submit --config=cloudbuild.yaml

# to build via commit
gcloud source repos create cosign-repo

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:cosign-kaniko@$PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/source.reader

gcloud source repos clone cosign-repo
cd cosign-repo
cp -R ../app/* .

git add -A
git commit -m "add"
git push 

# create a manual trigger
gcloud beta builds triggers create manual --region=global \
   --name=cosign-trigger --build-config=cloudbuild.yaml \
   --repo=https://source.developers.google.com/p/$PROJECT_ID/r/cosign-repo \
   --repo-type=CLOUD_SOURCE_REPOSITORIES --branch=main \
   --service-account=projects/$PROJECT_ID/serviceAccounts/cosign-kaniko@$PROJECT_ID.iam.gserviceaccount.com 

# now trigger
gcloud beta builds triggers run cosign-trigger --branch=main
```

![images/artifacts.png](images/artifacts.png)

### Verify

We are now ready to verify the images locally and using `cosign`


#### KMS

For kms keys, verify by either downloading kms public key

```bash
cd ../
gcloud kms keys versions get-public-key 1  \
  --key=key1 --keyring=cosignkr \
  --location=global --output-file=kms_pub.pem


# verify using the local key 
cosign verify --key kms_pub.pem   \
   us-central1-docker.pkg.dev/$PROJECT_ID/repo1/securebuild-kaniko@sha256:74110f406e1ad3d56301a9bb68f93118f0c8dc3170a5e114a96f7db76ad30225  | jq '.'

# or by api
cosign verify --key gcpkms://projects/$PROJECT_ID/locations/global/keyRings/cosignkr/cryptoKeys/key1/cryptoKeyVersions/1 \
      us-central1-docker.pkg.dev/$PROJECT_ID/repo1/securebuild-kaniko@sha256:74110f406e1ad3d56301a9bb68f93118f0c8dc3170a5e114a96f7db76ad30225 | jq '.'

Verification for us-central1-docker.pkg.dev/$PROJECT_ID/repo1/securebuild-kaniko@sha256:74110f406e1ad3d56301a9bb68f93118f0c8dc3170a5e114a96f7db76ad30225 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key
[
  {
    "critical": {
      "identity": {
        "docker-reference": "us-central1-docker.pkg.dev/$PROJECT_ID/repo1/securebuild-kaniko"
      },
      "image": {
        "docker-manifest-digest": "sha256:74110f406e1ad3d56301a9bb68f93118f0c8dc3170a5e114a96f7db76ad30225"
      },
      "type": "cosign container image signature"
    },
    "optional": {
      "key1": "value1"
    }
  }
]

```

Verify against transparency log checks

```bash
COSIGN_EXPERIMENTAL=1  cosign verify  us-central1-docker.pkg.dev/$PROJECT_ID/repo1/securebuild-kaniko@sha256:74110f406e1ad3d56301a9bb68f93118f0c8dc3170a5e114a96f7db76ad30225 | jq '.'


Verification for us-central1-docker.pkg.dev/$PROJECT_ID/repo1/securebuild-kaniko@sha256:74110f406e1ad3d56301a9bb68f93118f0c8dc3170a5e114a96f7db76ad30225 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - Any certificates were verified against the Fulcio roots.
[
  {
    "critical": {
      "identity": {
        "docker-reference": "us-central1-docker.pkg.dev/$PROJECT_ID/repo1/securebuild-kaniko"
      },
      "image": {
        "docker-manifest-digest": "sha256:74110f406e1ad3d56301a9bb68f93118f0c8dc3170a5e114a96f7db76ad30225"
      },
      "type": "cosign container image signature"
    },
    "optional": {
      "1.3.6.1.4.1.57264.1.1": "https://accounts.google.com",
      "Bundle": {
        "SignedEntryTimestamp": "MEUCIQDa+6EH9waIbsEEfzeiZjhKruwbgDufcRJkxm73X6+GrgIgYbj4Iqd9iWZyk0lFURJz8cmZSPmKt7PeN3ipmRj7MNE=",
        "Payload": {
          "body": "eyJhcGlWZXJzaW9uIjoiMC4wLjEiLCJraW5kIjoiaGFzaGVkcmVrb3JkIiwic3BlYyI6eyJkYXRhIjp7Imhhc2giOnsiYWxnb3JpdGhtIjoic2hhMjU2IiwidmFsdWUiOiJmNDlkZWIyY2U0ZmNlYWY3ZWQzMWRhNDNlOTI5NjgzYzE1MTM5NThiMWFlMjBiZmNkOTBkZTVmMTBjYTZlNTE2In19LCJzaWduYXR1cmUiOnsiY29udGVudCI6Ik1FUUNJRWhtWkJ3VmhTWlZzMkJHWFF6OGhUMnppb2RvaEw4Nk5TMjVpOUlSZjRjcUFpQjJQdDQ5Ty9WeXl0clNBUjVmWXB3TlYvWU9ZZEsrbnhUQjRsTWNSOGZCK1E9PSIsInB1YmxpY0tleSI6eyJjb250ZW50IjoiTFMwdExTMUNSVWRKVGlCRFJWSlVTVVpKUTBGVVJTMHRMUzB0Q2sxSlNVTTRha05EUVc1dFowRjNTVUpCWjBsVlNrdGFjRkJtVVVsbU4zVmtiMEozVkRCUWRFWldZbVZEWlhFNGQwTm5XVWxMYjFwSmVtb3dSVUYzVFhjS1RucEZWazFDVFVkQk1WVkZRMmhOVFdNeWJHNWpNMUoyWTIxVmRWcEhWakpOVWpSM1NFRlpSRlpSVVVSRmVGWjZZVmRrZW1SSE9YbGFVekZ3WW01U2JBcGpiVEZzV2tkc2FHUkhWWGRJYUdOT1RXcE5kMDVVUVRSTmFrVXhUMVJCZDFkb1kwNU5hazEzVGxSQk5FMXFTWGRQVkVGM1YycEJRVTFHYTNkRmQxbElDa3R2V2tsNmFqQkRRVkZaU1V0dldrbDZhakJFUVZGalJGRm5RVVZNWnpCUU5sTnBWbTQzVUcxclNrNXRaRWRsVTNWUldVWnNaRlpSWmtWSlF6VnlOMUFLYms4M1pqZDNkWEp5T1U5UGJrOWlNMUEzZDA1VU9VSjJkeXRNS3l0clJXdHpTazVTVTNGeGNWRXdVVzFKU0VkNGJXRlBRMEZhWjNkblowZFZUVUUwUndwQk1WVmtSSGRGUWk5M1VVVkJkMGxJWjBSQlZFSm5UbFpJVTFWRlJFUkJTMEpuWjNKQ1owVkdRbEZqUkVGNlFXUkNaMDVXU0ZFMFJVWm5VVlZ2Y1ZRMUNsQnpkRE5sVEhGQmNWaDRjVWN4U25aSmIwaHBVVmhuZDBoM1dVUldVakJxUWtKbmQwWnZRVlV6T1ZCd2VqRlphMFZhWWpWeFRtcHdTMFpYYVhocE5Ga0tXa1E0ZDFKM1dVUldVakJTUVZGSUwwSkVNSGRQTkVVMVdUSTVlbUZYWkhWTVYzUm9ZbTFzY21Jd1FuUmhWelZzWTIxR2MweFhNWEJpYmxZd1lWZEZkQXBQUkVsM1RHMXNhR0pUTlc1ak1sWjVaRzFzYWxwWFJtcFpNamt4WW01UmRWa3lPWFJOUTJ0SFEybHpSMEZSVVVKbk56aDNRVkZGUlVjeWFEQmtTRUo2Q2s5cE9IWlpWMDVxWWpOV2RXUklUWFZhTWpsMldqSjRiRXh0VG5aaVZFRnlRbWR2Y2tKblJVVkJXVTh2VFVGRlNVSkNNRTFITW1nd1pFaENlazlwT0hZS1dWZE9hbUl6Vm5Wa1NFMTFXakk1ZGxveWVHeE1iVTUyWWxSRFFtbDNXVXRMZDFsQ1FrRklWMlZSU1VWQloxSTVRa2h6UVdWUlFqTkJUakE1VFVkeVJ3cDRlRVY1V1hoclpVaEtiRzVPZDB0cFUydzJORE5xZVhRdk5HVkxZMjlCZGt0bE5rOUJRVUZDYUM4eFowRmpSVUZCUVZGRVFVVm5kMUpuU1doQlMweDBDazg1ZVdWSVFUaFFORFoxY2tJMlJqSXdWbXhZUlVSdlMyZGhPVzFyY2psalNtSlFXa0ZTTmxwQmFVVkJja0pTV1d4Wk1FaG1OM2hYZGk5S1QxWlRjek1LV25WNGQzRkpkVTlXVmxCemIxazJRWE0yV1haWldqQjNRMmRaU1V0dldrbDZhakJGUVhkTlJGcDNRWGRhUVVsM1RFWXJSRlZNZUdGUmNEVkNVWHBXTmdwNlprOTNLMnhFUVRBck1tSlRNVWsyY1hkV1RtbEVSelo1ZHl0aFV5c3pSVWhvZVRCTEwwTklWVEJEY2tkalZVWkJha0Z5Y1VaeGIzYzJOamR6YzI1TkNqTm9jakJRZVU5elpUaEZTeTgzVjJSR1NVSlBiR2RaVmxwVlpIZFdabEZyTDJ4bFltODBNREI0WldSb2FVdFFaRWQwUlQwS0xTMHRMUzFGVGtRZ1EwVlNWRWxHU1VOQlZFVXRMUzB0TFFvPSJ9fX19",
          "integratedTime": 1683583141,
          "logIndex": 20059298,
          "logID": "c0d23d6ad406973f9559f3ba2d1ca01f84147d8ffc5b8445c224f98b9591801d"
        }
      },
      "Issuer": "https://accounts.google.com",
      "Subject": "cosign-kaniko@$PROJECT_ID.iam.gserviceaccount.com",
      "key1": "value1"
    }
  }
]
```

payload decodes to

```json
{
  "apiVersion": "0.0.1",
  "kind": "hashedrekord",
  "spec": {
    "data": {
      "hash": {
        "algorithm": "sha256",
        "value": "f49deb2ce4fceaf7ed31da43e929683c1513958b1ae20bfcd90de5f10ca6e516"
      }
    },
    "signature": {
      "content": "MEQCIEhmZBwVhSZVs2BGXQz8hT2ziodohL86NS25i9IRf4cqAiB2Pt49O/VyytrSAR5fYpwNV/YOYdK+nxTB4lMcR8fB+Q==",
      "publicKey": {
        "content": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM4akNDQW5tZ0F3SUJBZ0lVSktacFBmUUlmN3Vkb0J3VDBQdEZWYmVDZXE4d0NnWUlLb1pJemowRUF3TXcKTnpFVk1CTUdBMVVFQ2hNTWMybG5jM1J2Y21VdVpHVjJNUjR3SEFZRFZRUURFeFZ6YVdkemRHOXlaUzFwYm5SbApjbTFsWkdsaGRHVXdIaGNOTWpNd05UQTRNakUxT1RBd1doY05Nak13TlRBNE1qSXdPVEF3V2pBQU1Ga3dFd1lICktvWkl6ajBDQVFZSUtvWkl6ajBEQVFjRFFnQUVMZzBQNlNpVm43UG1rSk5tZEdlU3VRWUZsZFZRZkVJQzVyN1AKbk83Zjd3dXJyOU9Pbk9iM1A3d05UOUJ2dytMKytrRWtzSk5SU3FxcVEwUW1JSEd4bWFPQ0FaZ3dnZ0dVTUE0RwpBMVVkRHdFQi93UUVBd0lIZ0RBVEJnTlZIU1VFRERBS0JnZ3JCZ0VGQlFjREF6QWRCZ05WSFE0RUZnUVVvcVQ1ClBzdDNlTHFBcVh4cUcxSnZJb0hpUVhnd0h3WURWUjBqQkJnd0ZvQVUzOVBwejFZa0VaYjVxTmpwS0ZXaXhpNFkKWkQ4d1J3WURWUjBSQVFIL0JEMHdPNEU1WTI5emFXZHVMV3RoYm1scmIwQnRhVzVsY21Gc0xXMXBiblYwYVdFdApPREl3TG1saGJTNW5jMlZ5ZG1salpXRmpZMjkxYm5RdVkyOXRNQ2tHQ2lzR0FRUUJnNzh3QVFFRUcyaDBkSEJ6Ck9pOHZZV05qYjNWdWRITXVaMjl2WjJ4bExtTnZiVEFyQmdvckJnRUVBWU8vTUFFSUJCME1HMmgwZEhCek9pOHYKWVdOamIzVnVkSE11WjI5dloyeGxMbU52YlRDQml3WUtLd1lCQkFIV2VRSUVBZ1I5QkhzQWVRQjNBTjA5TUdyRwp4eEV5WXhrZUhKbG5Od0tpU2w2NDNqeXQvNGVLY29BdktlNk9BQUFCaC8xZ0FjRUFBQVFEQUVnd1JnSWhBS0x0Ck85eWVIQThQNDZ1ckI2RjIwVmxYRURvS2dhOW1rcjljSmJQWkFSNlpBaUVBckJSWWxZMEhmN3hXdi9KT1ZTczMKWnV4d3FJdU9WVlBzb1k2QXM2WXZZWjB3Q2dZSUtvWkl6ajBFQXdNRFp3QXdaQUl3TEYrRFVMeGFRcDVCUXpWNgp6Zk93K2xEQTArMmJTMUk2cXdWTmlERzZ5dythUyszRUhoeTBLL0NIVTBDckdjVUZBakFycUZxb3c2Njdzc25NCjNocjBQeU9zZThFSy83V2RGSUJPbGdZVlpVZHdWZlFrL2xlYm80MDB4ZWRoaUtQZEd0RT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo="
      }
    }
  }
}
```


Then use `hashedrekord`: `f49deb2ce4fceaf7ed31da43e929683c1513958b1ae20bfcd90de5f10ca6e516` with `rekor-cli`

```bash
rekor-cli search --rekor_server https://rekor.sigstore.dev \
   --sha  f49deb2ce4fceaf7ed31da43e929683c1513958b1ae20bfcd90de5f10ca6e516
    Found matching entries (listed by UUID):
    24296fb24b8ad77a00097d2cc4ec9c73f93a2488fe08d4b81a1506dd12a1227e27234ee35067c3b5

rekor-cli search --rekor_server https://rekor.sigstore.dev  --email cosign-kaniko@$PROJECT_ID.iam.gserviceaccount.com

Found matching entries (listed by UUID):
24296fb24b8ad77a73b9d4b6cecdc582d03894c3c2e70bc4495c097d71dff6d18d583efbfd484816
24296fb24b8ad77a00097d2cc4ec9c73f93a2488fe08d4b81a1506dd12a1227e27234ee35067c3b5
24296fb24b8ad77aa2812ee2e2bbc2af2858456eca9d42ec199ce5c0e3355785c23555acd1fc869d
```


custom predicate

```bash
rekor-cli get --rekor_server https://rekor.sigstore.dev \
   --uuid 24296fb24b8ad77a73b9d4b6cecdc582d03894c3c2e70bc4495c097d71dff6d18d583efbfd484816

LogID: c0d23d6ad406973f9559f3ba2d1ca01f84147d8ffc5b8445c224f98b9591801d
Attestation: {"_type":"https://in-toto.io/Statement/v0.1","predicateType":"cosign.sigstore.dev/attestation/v1","subject":[{"name":"us-central1-docker.pkg.dev/$PROJECT_ID/repo1/securebuild-kaniko","digest":{"sha256":"74110f406e1ad3d56301a9bb68f93118f0c8dc3170a5e114a96f7db76ad30225"}}],"predicate":{"Data":"{ \"projectid\": \"$PROJECT_ID\", \"buildid\": \"680f4a05-4255-4af1-8a36-03ab849086be\", \"foo\":\"bar\", \"commitsha\": \"12fa2715e8a607ecbf20bede31765530f9b092f9\", \"name_hash\": \"$(cat /workspace/name_hash.txt)\"}","Timestamp":"2023-05-08T21:59:04Z"}}
Index: 20059304
IntegratedTime: 2023-05-08T21:59:05Z
UUID: 24296fb24b8ad77a73b9d4b6cecdc582d03894c3c2e70bc4495c097d71dff6d18d583efbfd484816
Body: {
  "IntotoObj": {
    "content": {
      "hash": {
        "algorithm": "sha256",
        "value": "562d0cbc077b780589f285f54afc7baf8c1e9ad8d67d03946ad6dba295e0255d"
      },
      "payloadHash": {
        "algorithm": "sha256",
        "value": "a2230f9bfde6f0a053e0c3c4dd18dfe97ff98daf767349aa48e73512a71b9a0f"
      }
    },
    "publicKey": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM4akNDQW5pZ0F3SUJBZ0lVRzJFTmlRL0NyZlJvK3kvcGwwNzlaSE1TSkFVd0NnWUlLb1pJemowRUF3TXcKTnpFVk1CTUdBMVVFQ2hNTWMybG5jM1J2Y21VdVpHVjJNUjR3SEFZRFZRUURFeFZ6YVdkemRHOXlaUzFwYm5SbApjbTFsWkdsaGRHVXdIaGNOTWpNd05UQTRNakUxT1RBMFdoY05Nak13TlRBNE1qSXdPVEEwV2pBQU1Ga3dFd1lICktvWkl6ajBDQVFZSUtvWkl6ajBEQVFjRFFnQUViUVFMa3c3a2J6ZEFIaTR4RTZzeU1WdkxmM3dnTk9BSkVWRGsKcnd3ZGIyUDF4Qml3SW9WOGxaS1lsSnEwdGt3RU9wSjNxQmhFV0srSGNkSUNmYnY2b0tPQ0FaY3dnZ0dUTUE0RwpBMVVkRHdFQi93UUVBd0lIZ0RBVEJnTlZIU1VFRERBS0JnZ3JCZ0VGQlFjREF6QWRCZ05WSFE0RUZnUVU3WXZtCmR2Zk5rLzMzQ20xY1dleG8vRmw5SU40d0h3WURWUjBqQkJnd0ZvQVUzOVBwejFZa0VaYjVxTmpwS0ZXaXhpNFkKWkQ4d1J3WURWUjBSQVFIL0JEMHdPNEU1WTI5emFXZHVMV3RoYm1scmIwQnRhVzVsY21Gc0xXMXBiblYwYVdFdApPREl3TG1saGJTNW5jMlZ5ZG1salpXRmpZMjkxYm5RdVkyOXRNQ2tHQ2lzR0FRUUJnNzh3QVFFRUcyaDBkSEJ6Ck9pOHZZV05qYjNWdWRITXVaMjl2WjJ4bExtTnZiVEFyQmdvckJnRUVBWU8vTUFFSUJCME1HMmgwZEhCek9pOHYKWVdOamIzVnVkSE11WjI5dloyeGxMbU52YlRDQmlnWUtLd1lCQkFIV2VRSUVBZ1I4QkhvQWVBQjJBTjA5TUdyRwp4eEV5WXhrZUhKbG5Od0tpU2w2NDNqeXQvNGVLY29BdktlNk9BQUFCaC8xZ0VoQUFBQVFEQUVjd1JRSWhBTXZKCllsRHFDLzJ2YXBCR0R1WXZlTmJmdDY2bG0yUHkwRzZsYXlPRUNQbWVBaUFuVmRlYjJBZ2dMNWhmR0ZsbkpCV0IKUFI1aTNQQTFEMVRNbFZXSUx0bHAwVEFLQmdncWhrak9QUVFEQXdOb0FEQmxBakI0UmR4Q20xak90ZUxwYnVDcwpUbTV5dWFNeEFKNFcrQk96ZGdPZGFNRlFmUXRhUG9SRzl0SG1QWVVDRndHZ0NGZ0NNUUNjWkp4MExWVWhIT2hICkZpaFNvT28ra0NYeXhseTl3N2lwcEJaaldQejg2Ujh0MTJQUzM4VUdPeXVpdDEvWTlGQT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo="
  }
}
```

```bash
rekor-cli get --rekor_server https://rekor.sigstore.dev \
   --uuid 24296fb24b8ad77a13b7e3f46e69cdb3b7ffef0b0d1c5e48e5cb5480e08ab1fa4528a9a0f1529f11     

LogID: c0d23d6ad406973f9559f3ba2d1ca01f84147d8ffc5b8445c224f98b9591801d
Index: 20059298
IntegratedTime: 2023-05-08T21:59:01Z
UUID: 24296fb24b8ad77a00097d2cc4ec9c73f93a2488fe08d4b81a1506dd12a1227e27234ee35067c3b5
Body: {
  "HashedRekordObj": {
    "data": {
      "hash": {
        "algorithm": "sha256",
        "value": "f49deb2ce4fceaf7ed31da43e929683c1513958b1ae20bfcd90de5f10ca6e516"
      }
    },
    "signature": {
      "content": "MEQCIEhmZBwVhSZVs2BGXQz8hT2ziodohL86NS25i9IRf4cqAiB2Pt49O/VyytrSAR5fYpwNV/YOYdK+nxTB4lMcR8fB+Q==",
      "publicKey": {
        "content": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM4akNDQW5tZ0F3SUJBZ0lVSktacFBmUUlmN3Vkb0J3VDBQdEZWYmVDZXE4d0NnWUlLb1pJemowRUF3TXcKTnpFVk1CTUdBMVVFQ2hNTWMybG5jM1J2Y21VdVpHVjJNUjR3SEFZRFZRUURFeFZ6YVdkemRHOXlaUzFwYm5SbApjbTFsWkdsaGRHVXdIaGNOTWpNd05UQTRNakUxT1RBd1doY05Nak13TlRBNE1qSXdPVEF3V2pBQU1Ga3dFd1lICktvWkl6ajBDQVFZSUtvWkl6ajBEQVFjRFFnQUVMZzBQNlNpVm43UG1rSk5tZEdlU3VRWUZsZFZRZkVJQzVyN1AKbk83Zjd3dXJyOU9Pbk9iM1A3d05UOUJ2dytMKytrRWtzSk5SU3FxcVEwUW1JSEd4bWFPQ0FaZ3dnZ0dVTUE0RwpBMVVkRHdFQi93UUVBd0lIZ0RBVEJnTlZIU1VFRERBS0JnZ3JCZ0VGQlFjREF6QWRCZ05WSFE0RUZnUVVvcVQ1ClBzdDNlTHFBcVh4cUcxSnZJb0hpUVhnd0h3WURWUjBqQkJnd0ZvQVUzOVBwejFZa0VaYjVxTmpwS0ZXaXhpNFkKWkQ4d1J3WURWUjBSQVFIL0JEMHdPNEU1WTI5emFXZHVMV3RoYm1scmIwQnRhVzVsY21Gc0xXMXBiblYwYVdFdApPREl3TG1saGJTNW5jMlZ5ZG1salpXRmpZMjkxYm5RdVkyOXRNQ2tHQ2lzR0FRUUJnNzh3QVFFRUcyaDBkSEJ6Ck9pOHZZV05qYjNWdWRITXVaMjl2WjJ4bExtTnZiVEFyQmdvckJnRUVBWU8vTUFFSUJCME1HMmgwZEhCek9pOHYKWVdOamIzVnVkSE11WjI5dloyeGxMbU52YlRDQml3WUtLd1lCQkFIV2VRSUVBZ1I5QkhzQWVRQjNBTjA5TUdyRwp4eEV5WXhrZUhKbG5Od0tpU2w2NDNqeXQvNGVLY29BdktlNk9BQUFCaC8xZ0FjRUFBQVFEQUVnd1JnSWhBS0x0Ck85eWVIQThQNDZ1ckI2RjIwVmxYRURvS2dhOW1rcjljSmJQWkFSNlpBaUVBckJSWWxZMEhmN3hXdi9KT1ZTczMKWnV4d3FJdU9WVlBzb1k2QXM2WXZZWjB3Q2dZSUtvWkl6ajBFQXdNRFp3QXdaQUl3TEYrRFVMeGFRcDVCUXpWNgp6Zk93K2xEQTArMmJTMUk2cXdWTmlERzZ5dythUyszRUhoeTBLL0NIVTBDckdjVUZBakFycUZxb3c2Njdzc25NCjNocjBQeU9zZThFSy83V2RGSUJPbGdZVlpVZHdWZlFrL2xlYm80MDB4ZWRoaUtQZEd0RT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo="
      }
    }
  }
}
```

attestation with sbom.  Note the kaniko attestation here includes the full cyclonedx output for **ALL** packages (including go packages)

```bash
rekor-cli get --rekor_server https://rekor.sigstore.dev \
   --uuid 24296fb24b8ad77aa2812ee2e2bbc2af2858456eca9d42ec199ce5c0e3355785c23555acd1fc869d         

LogID: c0d23d6ad406973f9559f3ba2d1ca01f84147d8ffc5b8445c224f98b9591801d
Attestation: {"_type":"https://in-toto.io/Statement/v0.1","predicateType":"https://cyclonedx.org/bom/v1.4","subject":[{"name":"us-central1-docker.pkg.dev/$PROJECT_ID/repo1/securebuild-kaniko","digest":{"sha256":"74110f406e1ad3d56301a9bb68f93118f0c8dc3170a5e114a96f7db76ad30225"}}],"predicate":{"bomFormat":"CycloneDX","components":[{"bom-ref":"pkg:deb/debian/base-files@11.1+deb11u7?arch=amd64\u0026distro=debian-11\u0026package-id=4046c56c645e8a80","cpe":"cpe:2.3:a:base-files:base-files:11.1\\+deb11u7:*:*:*:*:*:*:*","licenses":[{"license":{"name":"GPL"}}],"name":"base-files","properties":[{"name":"syft:package:foundBy","value":"dpkgdb-cataloger"},{"name":"syft:package:metadataType","value":"DpkgMetadata"},{"name":"syft:package:type","value":"deb"},{"name":"syft:cpe23","value":"cpe:2.3:a:base-files:base_files:11.1\\+deb11u7:*:*:*:*:*:*:*"},{"name":"syft:cpe23","value":"cpe:2.3:a:base_files:base-files:11.1\\+deb11u7:*:*:*:*:*:*:*"},{"name":"syft:cpe23","value":"cpe:2.3:a:base_files:base_files:11.1\\+deb11u7:*:*:*:*:*:*:*"},{"name":"syft:cpe23","value":"cpe:2.3:a:base:base-files:11.1\\+deb11u7:*:*:*:*:*:*:*"},{"name":"syft:cpe23","value":"cpe:2.3:a:base:base_files:11.1\\+deb11u7:*:*:*:*:*:*:*"},{"name":"syft:location:0:layerID","value":"sha256:f5a03f5a38794f03ae030b7eb72bc5ff8b5ba5b0e5b02e63e034949295a3cacf"},{"name":"syft:location:0:path","value":"/usr/share/doc/base-files/copyright"},{"name":"syft:location:1:layerID","value":"sha256:f5a03f5a38794f03ae030b7eb72bc5ff8b5ba5b0e5b02e63e034949295a3cacf"},{"name":"syft:location:1:path","value":"/var/lib/dpkg/status.d/base-files"},{"name":"syft:metadata:installedSize","value":"341"}],"publisher":"Santiago Vila \u003csanvila@debian.org\u003e","purl":"pkg:deb/debian/base-files@11.1+deb11u7?arch=amd64\u0026distro=debian-11","type":"library","version":"11.1+deb11u7"},{"bom-ref":"pkg:golang/github.com/gorilla/mux@v1.8.0?package-id=aabec3790e138139","cpe":"cpe:2.3:a:gorilla:mux:v1.8.0:*:*:*:*:*:*:*","name":"github.com/gorilla/mux","properties":[{"name":"syft:package:foundBy","value":"go-module-binary-cataloger"},{"name":"syft:package:language","value":"go"},{"name":"syft:package:metadataType","value":"GolangBinMetadata"},{"name":"syft:package:type","value":"go-module"},{"name":"syft:location:0:layerID","value":"sha256:8916294fdda674ecb67bcfd6729045679baa6e8c79a23fac9ccde33fd5a5df89"},{"name":"syft:location:0:path","value":"/home/nonroot/server"},{"name":"syft:metadata:architecture","value":"amd64"},{"name":"syft:metadata:goCompiledVersion","value":"go1.19.8"},{"name":"syft:metadata:h1Digest","value":"h1:i40aqfkR1h2SlN9hojwV5ZA91wcXFOvkdNIeFDP5koI="},{"name":"syft:metadata:mainModule","value":"github.com/salrashid123/cosign_bazel_cloud_build/app"}],"purl":"pkg:golang/github.com/gorilla/mux@v1.8.0","type":"library","version":"v1.8.0"},{"bom-ref":"pkg:golang/github.com/salrashid123/cosign_bazel_cloud_build/app@(devel)?package-id=81730e388144e355","cpe":"cpe:2.3:a:salrashid123:cosign-bazel-cloud-build\\/app:\\(devel\\):*:*:*:*:*:*:*","name":"github.com/salrashid123/cosign_bazel_cloud_build/app","properties":[{"name":"syft:package:foundBy","value":"go-module-binary-cataloger"},{"name":"syft:package:language","value":"go"},{"name":"syft:package:metadataType","value":"GolangBinMetadata"},{"name":"syft:package:type","value":"go-module"},{"name":"syft:cpe23","value":"cpe:2.3:a:salrashid123:cosign_bazel_cloud_build\\/app:\\(devel\\):*:*:*:*:*:*:*"},{"name":"syft:location:0:layerID","value":"sha256:8916294fdda674ecb67bcfd6729045679baa6e8c79a23fac9ccde33fd5a5df89"},{"name":"syft:location:0:path","value":"/home/nonroot/server"},{"name":"syft:metadata:architecture","value":"amd64"},{"name":"syft:metadata:goBuildSettings:-compiler","value":"gc"},{"name":"syft:metadata:goBuildSettings:CGO_ENABLED","value":"1"},{"name":"syft:metadata:goBuildSettings:GOAMD64","value":"v1"},{"name":"syft:metadata:goBuildSettings:GOARCH","value":"amd64"},{"name":"syft:metadata:goBuildSettings:GOOS","value":"linux"},{"name":"syft:metadata:goCompiledVersion","value":"go1.19.8"},{"name":"syft:metadata:mainModule","value":"github.com/salrashid123/cosign_bazel_cloud_build/app"}],"purl":"pkg:golang/github.com/salrashid123/cosign_bazel_cloud_build/app@(devel)","type":"library","version":"(devel)"},{"bom-ref":"pkg:golang/golang.org/x/net@v0.0.0-20220921203646-d300de134e69?package-id=74b6670eaea262e9","cpe":"cpe:2.3:a:golang:x\\/net:v0.0.0-20220921203646-d300de134e69:*:*:*:*:*:*:*","name":"golang.org/x/net","properties":[{"name":"syft:package:foundBy","value":"go-module-binary-cataloger"},{"name":"syft:package:language","value":"go"},{"name":"syft:package:metadataType","value":"GolangBinMetadata"},{"name":"syft:package:type","value":"go-module"},{"name":"syft:location:0:layerID","value":"sha256:8916294fdda674ecb67bcfd6729045679baa6e8c79a23fac9ccde33fd5a5df89"},{"name":"syft:location:0:path","value":"/home/nonroot/server"},{"name":"syft:metadata:architecture","value":"amd64"},{"name":"syft:metadata:goCompiledVersion","value":"go1.19.8"},{"name":"syft:metadata:h1Digest","value":"h1:hUJpGDpnfwdJW8iNypFjmSY0sCBEL+spFTZ2eO+Sfps="},{"name":"syft:metadata:mainModule","value":"github.com/salrashid123/cosign_bazel_cloud_build/app"}],"purl":"pkg:golang/golang.org/x/net@v0.0.0-20220921203646-d300de134e69","type":"library","version":"v0.0.0-20220921203646-d300de134e69"},{"bom-ref":"pkg:golang/golang.org/x/text@v0.3.7?package-id=17229fdd952f28a1","cpe":"cpe:2.3:a:golang:x\\/text:v0.3.7:*:*:*:*:*:*:*","name":"golang.org/x/text","properties":[{"name":"syft:package:foundBy","value":"go-module-binary-cataloger"},{"name":"syft:package:language","value":"go"},{"name":"syft:package:metadataType","value":"GolangBinMetadata"},{"name":"syft:package:type","value":"go-module"},{"name":"syft:location:0:layerID","value":"sha256:8916294fdda674ecb67bcfd6729045679baa6e8c79a23fac9ccde33fd5a5df89"},{"name":"syft:location:0:path","value":"/home/nonroot/server"},{"name":"syft:metadata:architecture","value":"amd64"},{"name":"syft:metadata:goCompiledVersion","value":"go1.19.8"},{"name":"syft:metadata:h1Digest","value":"h1:olpwvP2KacW1ZWvsR7uQhoyTYvKAupfQrRGBFM352Gk="},{"name":"syft:metadata:mainModule","value":"github.com/salrashid123/cosign_bazel_cloud_build/app"}],"purl":"pkg:golang/golang.org/x/text@v0.3.7","type":"library","version":"v0.3.7"},{"bom-ref":"pkg:deb/debian/libc6@2.31-13+deb11u6?arch=amd64\u0026upstream=glibc\u0026distro=debian-11\u0026package-id=63c990ac96615753","cpe":"cpe:2.3:a:libc6:libc6:2.31-13\\+deb11u6:*:*:*:*:*:*:*","licenses":[{"license":{"id":"GPL-2.0-only"}},{"license":{"id":"LGPL-2.1-only"}}],"name":"libc6","properties":[{"name":"syft:package:foundBy","value":"dpkgdb-cataloger"},{"name":"syft:package:metadataType","value":"DpkgMetadata"},{"name":"syft:package:type","value":"deb"},{"name":"syft:location:0:layerID","value":"sha256:4696325a830059c4a641fc26f4191f12c06067bc67afc8eaf26ab1f5c957fabb"},{"name":"syft:location:0:path","value":"/usr/share/doc/libc6/copyright"},{"name":"syft:location:1:layerID","value":"sha256:4696325a830059c4a641fc26f4191f12c06067bc67afc8eaf26ab1f5c957fabb"},{"name":"syft:location:1:path","value":"/var/lib/dpkg/status.d/libc6"},{"name":"syft:metadata:installedSize","value":"12833"},{"name":"syft:metadata:source","value":"glibc"}],"publisher":"GNU Libc Maintainers \u003cdebian-glibc@lists.debian.org\u003e","purl":"pkg:deb/debian/libc6@2.31-13+deb11u6?arch=amd64\u0026upstream=glibc\u0026distro=debian-11","type":"library","version":"2.31-13+deb11u6"},{"bom-ref":"pkg:deb/debian/libssl1.1@1.1.1n-0+deb11u4?arch=amd64\u0026upstream=openssl\u0026distro=debian-11\u0026package-id=442c38ad06093eaf","cpe":"cpe:2.3:a:libssl1.1:libssl1.1:1.1.1n-0\\+deb11u4:*:*:*:*:*:*:*","name":"libssl1.1","properties":[{"name":"syft:package:foundBy","value":"dpkgdb-cataloger"},{"name":"syft:package:metadataType","value":"DpkgMetadata"},{"name":"syft:package:type","value":"deb"},{"name":"syft:location:0:layerID","value":"sha256:ffd6e7a11a7918016b61241aae9af686e52078f9ca59905c02c770cd2b64f7f4"},{"name":"syft:location:0:path","value":"/usr/share/doc/libssl1.1/copyright"},{"name":"syft:location:1:layerID","value":"sha256:ffd6e7a11a7918016b61241aae9af686e52078f9ca59905c02c770cd2b64f7f4"},{"name":"syft:location:1:path","value":"/var/lib/dpkg/status.d/libssl1.1"},{"name":"syft:metadata:installedSize","value":"4124"},{"name":"syft:metadata:source","value":"openssl"}],"publisher":"Debian OpenSSL Team \u003cpkg-openssl-devel@lists.alioth.debian.org\u003e","purl":"pkg:deb/debian/libssl1.1@1.1.1n-0+deb11u4?arch=amd64\u0026upstream=openssl\u0026distro=debian-11","type":"library","version":"1.1.1n-0+deb11u4"},{"bom-ref":"pkg:deb/debian/netbase@6.3?arch=all\u0026distro=debian-11\u0026package-id=fdbf312837579aa","cpe":"cpe:2.3:a:netbase:netbase:6.3:*:*:*:*:*:*:*","licenses":[{"license":{"id":"GPL-2.0-only"}}],"name":"netbase","properties":[{"name":"syft:package:foundBy","value":"dpkgdb-cataloger"},{"name":"syft:package:metadataType","value":"DpkgMetadata"},{"name":"syft:package:type","value":"deb"},{"name":"syft:location:0:layerID","value":"sha256:375c3f212997db8f30df665af4c80bb2597ac491dc13fef36fe0213d035d4369"},{"name":"syft:location:0:path","value":"/usr/share/doc/netbase/copyright"},{"name":"syft:location:1:layerID","value":"sha256:375c3f212997db8f30df665af4c80bb2597ac491dc13fef36fe0213d035d4369"},{"name":"syft:location:1:path","value":"/var/lib/dpkg/status.d/netbase"},{"name":"syft:metadata:installedSize","value":"41"}],"publisher":"Marco d'Itri \u003cmd@linux.it\u003e","purl":"pkg:deb/debian/netbase@6.3?arch=all\u0026distro=debian-11","type":"library","version":"6.3"},{"bom-ref":"pkg:deb/debian/openssl@1.1.1n-0+deb11u4?arch=amd64\u0026distro=debian-11\u0026package-id=4d0d28254323ed2d","cpe":"cpe:2.3:a:openssl:openssl:1.1.1n-0\\+deb11u4:*:*:*:*:*:*:*","name":"openssl","properties":[{"name":"syft:package:foundBy","value":"dpkgdb-cataloger"},{"name":"syft:package:metadataType","value":"DpkgMetadata"},{"name":"syft:package:type","value":"deb"},{"name":"syft:location:0:layerID","value":"sha256:8ebb2609b2f005f8af918ad58220fcea53ccd9767bc45f63cf852df6cf968cf3"},{"name":"syft:location:0:path","value":"/usr/share/doc/openssl/copyright"},{"name":"syft:location:1:layerID","value":"sha256:8ebb2609b2f005f8af918ad58220fcea53ccd9767bc45f63cf852df6cf968cf3"},{"name":"syft:location:1:path","value":"/var/lib/dpkg/status.d/openssl"},{"name":"syft:metadata:installedSize","value":"1466"}],"publisher":"Debian OpenSSL Team \u003cpkg-openssl-devel@lists.alioth.debian.org\u003e","purl":"pkg:deb/debian/openssl@1.1.1n-0+deb11u4?arch=amd64\u0026distro=debian-11","type":"library","version":"1.1.1n-0+deb11u4"},{"bom-ref":"pkg:deb/debian/tzdata@2021a-1+deb11u10?arch=all\u0026distro=debian-11\u0026package-id=a257ef3c84e6537f","cpe":"cpe:2.3:a:tzdata:tzdata:2021a-1\\+deb11u10:*:*:*:*:*:*:*","name":"tzdata","properties":[{"name":"syft:package:foundBy","value":"dpkgdb-cataloger"},{"name":"syft:package:metadataType","value":"DpkgMetadata"},{"name":"syft:package:type","value":"deb"},{"name":"syft:location:0:layerID","value":"sha256:b20f821286a6a7701d9f75c376011fb4cede06ee40ee2a9152f51d2ff975a42b"},{"name":"syft:location:0:path","value":"/usr/share/doc/tzdata/copyright"},{"name":"syft:location:1:layerID","value":"sha256:b20f821286a6a7701d9f75c376011fb4cede06ee40ee2a9152f51d2ff975a42b"},{"name":"syft:location:1:path","value":"/var/lib/dpkg/status.d/tzdata"},{"name":"syft:metadata:installedSize","value":"3413"}],"publisher":"GNU Libc Maintainers \u003cdebian-glibc@lists.debian.org\u003e","purl":"pkg:deb/debian/tzdata@2021a-1+deb11u10?arch=all\u0026distro=debian-11","type":"library","version":"2021a-1+deb11u10"},{"description":"Distroless","externalReferences":[{"type":"issue-tracker","url":"https://github.com/GoogleContainerTools/distroless/issues/new"},{"type":"website","url":"https://github.com/GoogleContainerTools/distroless"},{"comment":"support","type":"other","url":"https://github.com/GoogleContainerTools/distroless/blob/master/README.md"}],"name":"debian","properties":[{"name":"syft:distro:id","value":"debian"},{"name":"syft:distro:prettyName","value":"Distroless"},{"name":"syft:distro:versionID","value":"11"}],"swid":{"name":"debian","tagId":"debian","version":"11"},"type":"operating-system","version":"11"}],"metadata":{"component":{"bom-ref":"97027fd1bee31de5","name":"us-central1-docker.pkg.dev/$PROJECT_ID/repo1/securebuild-kaniko@sha256:74110f406e1ad3d56301a9bb68f93118f0c8dc3170a5e114a96f7db76ad30225","type":"container","version":"sha256:8bdb6fa5c3e66714d248bcadaf124f970046d753d11c3725907164bc75371bc5"},"timestamp":"2023-05-08T21:58:58Z","tools":[{"name":"syft","vendor":"anchore","version":"0.78.0"}]},"serialNumber":"urn:uuid:ed92fb59-3918-46fb-a797-0c3f78ad3356","specVersion":"1.4","version":1}}
Index: 20059297
IntegratedTime: 2023-05-08T21:59:01Z
UUID: 24296fb24b8ad77aa2812ee2e2bbc2af2858456eca9d42ec199ce5c0e3355785c23555acd1fc869d
Body: {
  "IntotoObj": {
    "content": {
      "hash": {
        "algorithm": "sha256",
        "value": "21bd92afd3eaa9318583e99d5143e20e56a007827cc0e41ba048e075f7e02f14"
      },
      "payloadHash": {
        "algorithm": "sha256",
        "value": "fbab2ab06dfb8935f2e0382df38a0818a2b30aea9cd6e11b29d0676a21b5ef24"
      }
    },
    "publicKey": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM4akNDQW5pZ0F3SUJBZ0lVQzlrTmx6N2publg1bXNBZDVUdGxORWE5bmlJd0NnWUlLb1pJemowRUF3TXcKTnpFVk1CTUdBMVVFQ2hNTWMybG5jM1J2Y21VdVpHVjJNUjR3SEFZRFZRUURFeFZ6YVdkemRHOXlaUzFwYm5SbApjbTFsWkdsaGRHVXdIaGNOTWpNd05UQTRNakUxT1RBd1doY05Nak13TlRBNE1qSXdPVEF3V2pBQU1Ga3dFd1lICktvWkl6ajBDQVFZSUtvWkl6ajBEQVFjRFFnQUVMRVdsN1o4TGRHSy9Tdlc1K3BCRFUvTnd6TlVoVHROdGRXMkUKWlpjNmVqSXlaUjBXYm9xYlVITzZyVVU5Nm5oU2MvS3J5NTRTOStXMG4wT2R6c21RbjZPQ0FaY3dnZ0dUTUE0RwpBMVVkRHdFQi93UUVBd0lIZ0RBVEJnTlZIU1VFRERBS0JnZ3JCZ0VGQlFjREF6QWRCZ05WSFE0RUZnUVVFaFV4CndiRHBFczNBWHVsdmpRc1VKMCtkYjA0d0h3WURWUjBqQkJnd0ZvQVUzOVBwejFZa0VaYjVxTmpwS0ZXaXhpNFkKWkQ4d1J3WURWUjBSQVFIL0JEMHdPNEU1WTI5emFXZHVMV3RoYm1scmIwQnRhVzVsY21Gc0xXMXBiblYwYVdFdApPREl3TG1saGJTNW5jMlZ5ZG1salpXRmpZMjkxYm5RdVkyOXRNQ2tHQ2lzR0FRUUJnNzh3QVFFRUcyaDBkSEJ6Ck9pOHZZV05qYjNWdWRITXVaMjl2WjJ4bExtTnZiVEFyQmdvckJnRUVBWU8vTUFFSUJCME1HMmgwZEhCek9pOHYKWVdOamIzVnVkSE11WjI5dloyeGxMbU52YlRDQmlnWUtLd1lCQkFIV2VRSUVBZ1I4QkhvQWVBQjJBTjA5TUdyRwp4eEV5WXhrZUhKbG5Od0tpU2w2NDNqeXQvNGVLY29BdktlNk9BQUFCaC8xZ0F1TUFBQVFEQUVjd1JRSWhBTU1lCnFRbHBOaUtKQkJIWXZTWXF4WUlEczZldCtIZGFuQVNXbDZIZEpmYTJBaUErQ1ZFRStYQkFwSllkbGUwKzUyY00KUDFKZ0xnakNXQWJ3OTVyL2ZNNXU0ekFLQmdncWhrak9QUVFEQXdOb0FEQmxBakVBdGlZd0VxSWc4OXV1R2FUWQpiaHFuVERqSmhFWGN6U0hUemtKM3U0UXFwT1NobjJLYzZ4SnpnemV2N0ZGOU1OaThBakJpNFhvdXRQRXlreGhtCmpXR2c2QVVlcmVqTy9qVHE2VWZiRjUyMWwwaStFZFlvZTA0eEhmckljSndOR3k1emNvTT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo="
  }
}

```

where in my case the public key from the rekor entry is

```
-----BEGIN CERTIFICATE-----
MIIC9DCCAnmgAwIBAgIUJQMCGVyfKxaLhl8ee233sk+qfX0wCgYIKoZIzj0EAwMw
NzEVMBMGA1UEChMMc2lnc3RvcmUuZGV2MR4wHAYDVQQDExVzaWdzdG9yZS1pbnRl
cm1lZGlhdGUwHhcNMjMwNDI1MTMxMTA0WhcNMjMwNDI1MTMyMTA0WjAAMFkwEwYH
KoZIzj0CAQYIKoZIzj0DAQcDQgAE8G1WKK9KAbUfvxPIFr2RJR2k1qIxgy33GVB0
ohbYmf5bWYNMHLVkxYU1AZOCBHqLdM8C2pgKO6qbYNTOBzpQQ6OCAZgwggGUMA4G
A1UdDwEB/wQEAwIHgDATBgNVHSUEDDAKBggrBgEFBQcDAzAdBgNVHQ4EFgQUlEHG
DrvDPy6No1qK4TuZh2cu8kswHwYDVR0jBBgwFoAU39Ppz1YkEZb5qNjpKFWixi4Y
ZD8wSAYDVR0RAQH/BD4wPIE6Y29zaWduQGNvc2lnbi10ZXN0LWthbmlrby0xLTM4
NDgxMy5pYW0uZ3NlcnZpY2VhY2NvdW50LmNvbTApBgorBgEEAYO/MAEBBBtodHRw
czovL2FjY291bnRzLmdvb2dsZS5jb20wKwYKKwYBBAGDvzABCAQdDBtodHRwczov
L2FjY291bnRzLmdvb2dsZS5jb20wgYoGCisGAQQB1nkCBAIEfAR6AHgAdgDdPTBq
xscRMmMZHhyZZzcCokpeuN48rf+HinKALynujgAAAYe4igDNAAAEAwBHMEUCIQC/
45K8Gx9eHp4DsxbyaxUgmzNP7kwJ9dAKwxxKFkfITgIgBK4A9lOUCFbOV5366ats
6mNHZYFcT6ribdplLPKQFAEwCgYIKoZIzj0EAwMDaQAwZgIxALvt2g2pPJgpDyZS
yDzEzdukmTRS4k8QKVauUiM+W/77x2LUD7vhYaOm5Kt2q5gCNQIxAKQQ2z2WVw+q
g+sNXVxM9w+HQ6hbvqdvte+rHfwSjiqCp63fo+oMF3+G49658qO3MA==
-----END CERTIFICATE-----
```


which decoded is:

```bash
openssl x509 -in cert.pem -noout -text

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            0b:d9:0d:97:3e:e3:9e:75:f9:9a:c0:1d:e5:3b:65:34:46:bd:9e:22
        Signature Algorithm: ecdsa-with-SHA384
        Issuer: O = sigstore.dev, CN = sigstore-intermediate
        Validity
            Not Before: May  8 21:59:00 2023 GMT
            Not After : May  8 22:09:00 2023 GMT
        Subject: 
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:2c:45:a5:ed:9f:0b:74:62:bf:4a:f5:b9:fa:90:
                    43:53:f3:70:cc:d5:21:4e:d3:6d:75:6d:84:65:97:
                    3a:7a:32:32:65:1d:16:6e:8a:9b:50:73:ba:ad:45:
                    3d:ea:78:52:73:f2:ab:cb:9e:12:f7:e5:b4:9f:43:
                    9d:ce:c9:90:9f
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature
            X509v3 Extended Key Usage: 
                Code Signing
            X509v3 Subject Key Identifier: 
                12:15:31:C1:B0:E9:12:CD:C0:5E:E9:6F:8D:0B:14:27:4F:9D:6F:4E
            X509v3 Authority Key Identifier: 
                DF:D3:E9:CF:56:24:11:96:F9:A8:D8:E9:28:55:A2:C6:2E:18:64:3F
            X509v3 Subject Alternative Name: critical
                email:cosign-kaniko@$PROJECT_ID.iam.gserviceaccount.com   <<<<<<<<<<<<<<<<<<<<<
            1.3.6.1.4.1.57264.1.1: 
                https://accounts.google.com
            1.3.6.1.4.1.57264.1.8: 
                ..https://accounts.google.com
            CT Precertificate SCTs: 
                Signed Certificate Timestamp:
                    Version   : v1 (0x0)
                    Log ID    : DD:3D:30:6A:C6:C7:11:32:63:19:1E:1C:99:67:37:02:
                                A2:4A:5E:B8:DE:3C:AD:FF:87:8A:72:80:2F:29:EE:8E
                    Timestamp : May  8 21:59:00.579 2023 GMT
                    Extensions: none
                    Signature : ecdsa-with-SHA256
                                30:45:02:21:00:C3:1E:A9:09:69:36:22:89:04:11:D8:
                                BD:26:2A:C5:82:03:B3:A7:AD:F8:77:5A:9C:04:96:97:
                                A1:DD:25:F6:B6:02:20:3E:09:51:04:F9:70:40:A4:96:
                                1D:95:ED:3E:E7:67:0C:3F:52:60:2E:08:C2:58:06:F0:
                                F7:9A:FF:7C:CE:6E:E3
    Signature Algorithm: ecdsa-with-SHA384
```


---

### Appendix
#### local kaniko build artifiact registry permissions


The following config allows you to use local docker kaniko to push to container registry

```bash
token=$(gcloud auth print-access-token)
docker_token=$(echo -n "gclouddockertoken:$token" | base64 | tr -d "\n")

cat > ~/.docker/config_kaniko.json <<- EOM
{
  "auths": {
    "gcr.io": {
      "auth": "$docker_token",
      "email": "not@val.id"
    },
    "us.gcr.io": {
      "auth": "$docker_token",
      "email": "not@val.id"
    },
    "us-central1-docker.pkg.dev": {
      "auth": "$docker_token",
      "email": "not@val.id"
    }
  }
}
EOM
```