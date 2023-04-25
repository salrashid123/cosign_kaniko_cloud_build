# Deterministic container hashes and container signing using Cosign, Kaniko and Google Cloud Build

A simple tutorial that generates consistent container image hashes using `kankko` and then signs provenance records using [cosign](https://github.com/sigstore/cosign) (Container Signing, Verification and Storage in an OCI registry).


this is identical to [Deterministic container hashes and container signing using Cosign, Bazel and Google Cloud Build](https://github.com/salrashid123/cosign_bazel_cloud_build) except ofcourse it uses [kaniko](https://github.com/GoogleContainerTools/kaniko) and its flag that enabled reproduceability:


- [Flag reproducible](https://github.com/GoogleContainerTools/kaniko#flag---reproducible)

```
--reproducible

Set this flag to strip timestamps out of the built image and make it reproducible.
```

This will create a consistent image hash:
 
* `securebuild-kaniko@sha256:5e065a75f1dd137db2a1d7a5aada2160668cb5cb705732d8877e2c362cfc5935`

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
gcloud iam service-accounts create cosign

# allow 'self impersonation' for cloud build service account
gcloud iam service-accounts add-iam-policy-binding cosign@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/iam.serviceAccountTokenCreator \
    --member "serviceAccount:cosign@$PROJECT_ID.iam.gserviceaccount.com"

# allow cloud build to write logs
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:cosign@$PROJECT_ID.iam.gserviceaccount.com  \
  --role=roles/logging.logWriter

# allow cloud build write access to artifact registry
gcloud artifacts repositories add-iam-policy-binding repo1 \
    --location=us-central1  \
    --member=serviceAccount:cosign@$PROJECT_ID.iam.gserviceaccount.com \
    --role=roles/artifactregistry.writer

# allow cloud build access to list KMS keys
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:cosign@$PROJECT_ID.iam.gserviceaccount.com  \
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
    --member=serviceAccount:cosign@$PROJECT_ID.iam.gserviceaccount.com \
    --role=roles/cloudkms.signer

# allow current gcloud user to view the public key
gcloud kms keys add-iam-policy-binding key1 \
    --keyring=cosignkr --location=global \
    --member=serviceAccount:cosign@$PROJECT_ID.iam.gserviceaccount.com  \
    --role=roles/cloudkms.publicKeyViewer

# create a temp bucket for cloud build and allow cloud build permissions to use it
gsutil mb gs://$PROJECT_ID\_cloudbuild
gsutil iam ch serviceAccount:cosign@$PROJECT_ID.iam.gserviceaccount.com:objectAdmin gs://$PROJECT_ID\_cloudbuild
```

### Build image

```bash
cd /app
gcloud beta builds submit --config=cloudbuild.yaml --machine-type=n1-highcpu-32

# optionally create the application sbom and sign it with the same cosign keypair
# goreleaser release --snapshot  --rm-dist 
## for github
## git tag v1.0.0
## git push origin --tags
## goreleaser release --rm-dist
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
   us-central1-docker.pkg.dev/$PROJECT_ID/repo1/securebuild@sha256:5e065a75f1dd137db2a1d7a5aada2160668cb5cb705732d8877e2c362cfc5935  | jq '.'

# or by api
cosign verify --key gcpkms://projects/$PROJECT_ID/locations/global/keyRings/cosignkr/cryptoKeys/key1/cryptoKeyVersions/1 \
      us-central1-docker.pkg.dev/$PROJECT_ID/repo1/securebuild@sha256:5e065a75f1dd137db2a1d7a5aada2160668cb5cb705732d8877e2c362cfc5935 | jq '.'

Verification for us-central1-docker.pkg.dev/cosign-test-kaniko-1-384813/repo1/securebuild@sha256:5e065a75f1dd137db2a1d7a5aada2160668cb5cb705732d8877e2c362cfc5935 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key
[
  {
    "critical": {
      "identity": {
        "docker-reference": "us-central1-docker.pkg.dev/cosign-test-kaniko-1-384813/repo1/securebuild"
      },
      "image": {
        "docker-manifest-digest": "sha256:5e065a75f1dd137db2a1d7a5aada2160668cb5cb705732d8877e2c362cfc5935"
      },
      "type": "cosign container image signature"
    },
    "optional": {
      "key1": "value1"
    }
  }
]



COSIGN_EXPERIMENTAL=1  cosign verify  us-central1-docker.pkg.dev/$PROJECT_ID/repo1/securebuild@sha256:5e065a75f1dd137db2a1d7a5aada2160668cb5cb705732d8877e2c362cfc5935 | jq '.'


rekor-cli search --rekor_server https://rekor.sigstore.dev \
   --sha  05066f8e20a18df98be5050789b8935891908a921e3aabeed2d0378f1c236d88
    Found matching entries (listed by UUID):
    24296fb24b8ad77ae55e70f2c85f119946322c47d38dde1409cf760a8011a5dd180b01309095cc3a


## note, ifyou wanted to use the project ID is used in this repo to upload to rekor, set export PROJECT_ID=cosign-test-kaniko
rekor-cli search --rekor_server https://rekor.sigstore.dev  --email cosign@$PROJECT_ID.iam.gserviceaccount.com

Found matching entries (listed by UUID):
24296fb24b8ad77a6802c1ef56671cfb7f1df54a56914eeb5f7618b0906459f0f45135eebc52e8e2
24296fb24b8ad77a13b7e3f46e69cdb3b7ffef0b0d1c5e48e5cb5480e08ab1fa4528a9a0f1529f11
24296fb24b8ad77a61fa48c55a19124687ede29f830a6478d0316128ed4785cefcbde5d1423275e8
```


custom predicate

```bash
rekor-cli get --rekor_server https://rekor.sigstore.dev \
   --uuid 24296fb24b8ad77a6802c1ef56671cfb7f1df54a56914eeb5f7618b0906459f0f45135eebc52e8e2

LogID: c0d23d6ad406973f9559f3ba2d1ca01f84147d8ffc5b8445c224f98b9591801d
Attestation: {"_type":"https://in-toto.io/Statement/v0.1","predicateType":"cosign.sigstore.dev/attestation/v1","subject":[{"name":"us-central1-docker.pkg.dev/cosign-test-kaniko-1/repo1/securebuild","digest":{"sha256":"5e065a75f1dd137db2a1d7a5aada2160668cb5cb705732d8877e2c362cfc5935"}}],"predicate":{"Data":"{ \"projectid\": \"cosign-test-kaniko-1\", \"buildid\": \"a08b8033-3a26-4c5e-8d80-73ca3238ddbe\", \"foo\":\"bar\", \"commitsha\": \"fc9286ba352fd0207dbf3d3bc0bad0198ab5d9ec\", \"name_hash\": \"$(cat /workspace/name_hash.txt)\"}","Timestamp":"2023-04-22T19:56:23Z"}}
Index: 18669807
IntegratedTime: 2023-04-22T19:56:23Z
UUID: 24296fb24b8ad77a868630095549dbe5dcbae06f9f447ba0c7817908ce1d782c93a4440226e15e29
Body: {
  "IntotoObj": {
    "content": {
      "hash": {
        "algorithm": "sha256",
        "value": "26a8ce09dac3d10b1834a4ef9294117ce666296501e0b5d14c28ce84addeec70"
      },
      "payloadHash": {
        "algorithm": "sha256",
        "value": "42d5f6c4712031e0b09f0004e2ead28458717b3db1ae77745a0d5bc5c6b36b7d"
      }
    },
    "publicKey": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM3VENDQW5PZ0F3SUJBZ0lVUGJwQTZCM1JnNUdSRTR5OVkzNlNXaXB2ZnZzd0NnWUlLb1pJemowRUF3TXcKTnpFVk1CTUdBMVVFQ2hNTWMybG5jM1J2Y21VdVpHVjJNUjR3SEFZRFZRUURFeFZ6YVdkemRHOXlaUzFwYm5SbApjbTFsWkdsaGRHVXdIaGNOTWpNd05ESXlNVGsxTmpJeldoY05Nak13TkRJeU1qQXdOakl6V2pBQU1Ga3dFd1lICktvWkl6ajBDQVFZSUtvWkl6ajBEQVFjRFFnQUV1STlDeEVxNUZ3WmpIYWliK3VMbTkyYitVdXZTREVTc1ZaVDEKOE9OdHd3UlpvZjlBY3N5TkpYT2RZMGt3aS94YkdFbmIyM1ljVnJqanplZVRnMFV3V3FPQ0FaSXdnZ0dPTUE0RwpBMVVkRHdFQi93UUVBd0lIZ0RBVEJnTlZIU1VFRERBS0JnZ3JCZ0VGQlFjREF6QWRCZ05WSFE0RUZnUVVET2dHCkJOaHp1SWhIT1dpL1RUYXA0bGhiYms4d0h3WURWUjBqQkJnd0ZvQVUzOVBwejFZa0VaYjVxTmpwS0ZXaXhpNFkKWkQ4d1FRWURWUjBSQVFIL0JEY3dOWUV6WTI5emFXZHVRR052YzJsbmJpMTBaWE4wTFd0aGJtbHJieTB4TG1saApiUzVuYzJWeWRtbGpaV0ZqWTI5MWJuUXVZMjl0TUNrR0Npc0dBUVFCZzc4d0FRRUVHMmgwZEhCek9pOHZZV05qCmIzVnVkSE11WjI5dloyeGxMbU52YlRBckJnb3JCZ0VFQVlPL01BRUlCQjBNRzJoMGRIQnpPaTh2WVdOamIzVnUKZEhNdVoyOXZaMnhsTG1OdmJUQ0Jpd1lLS3dZQkJBSFdlUUlFQWdSOUJIc0FlUUIzQU4wOU1Hckd4eEV5WXhrZQpISmxuTndLaVNsNjQzanl0LzRlS2NvQXZLZTZPQUFBQmg2cUtBQlFBQUFRREFFZ3dSZ0loQUxFWDdWbC8xNEhGCkcvcEx5QWpHNjNRZHR0YWtqVHZBQk1DVS9RVEdidzlxQWlFQTFoVEFHRzZ6WUg5dmo0SXJLVzZlODk2c1FVdHoKTURtb05ZVVVEbEpKekxvd0NnWUlLb1pJemowRUF3TURhQUF3WlFJd0cvT2ZPYmFkT1paNEZmNFJnbFJxZU53cQpzczh3dldma0p5a3hMdldYRTk4cS9YcVVmUTJRcFA3MENvdnZLR2kzQWpFQXArYStnSXdCK01oSzJrbkpCZm5ZCm11NEFmR1RzeGhZZmlCeFdrMHdUTmNaMmo5YU9KNFk3cjlUZXV5RXVSeTRpCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K"
  }
}
```

```bash
rekor-cli get --rekor_server https://rekor.sigstore.dev \
   --uuid 24296fb24b8ad77a13b7e3f46e69cdb3b7ffef0b0d1c5e48e5cb5480e08ab1fa4528a9a0f1529f11     

LogID: c0d23d6ad406973f9559f3ba2d1ca01f84147d8ffc5b8445c224f98b9591801d
Index: 18669804
IntegratedTime: 2023-04-22T19:56:21Z
UUID: 24296fb24b8ad77abac2456499de9294a85a697e4ef17ae8d2693aa6b7e875f89745044385131e1c
Body: {
  "HashedRekordObj": {
    "data": {
      "hash": {
        "algorithm": "sha256",
        "value": "3bd4831711b3e3153576b2346a57a5ceba140ae3188c85e864b726efd1eff0ee"
      }
    },
    "signature": {
      "content": "MEUCIQC4PoWR3h1QA0NMe8CinRPfeqUQdURbd8MH+wBo5jH+SwIgd9uRpFJZ3QyUSpHslVrnnWBL0+Bk/B6roi4Z6gA/fGM=",
      "publicKey": {
        "content": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM3RENDQW5LZ0F3SUJBZ0lVWkI5ejdiWVlhQ2R6ZGdSc054NzAwZnFwdHNNd0NnWUlLb1pJemowRUF3TXcKTnpFVk1CTUdBMVVFQ2hNTWMybG5jM1J2Y21VdVpHVjJNUjR3SEFZRFZRUURFeFZ6YVdkemRHOXlaUzFwYm5SbApjbTFsWkdsaGRHVXdIaGNOTWpNd05ESXlNVGsxTmpJd1doY05Nak13TkRJeU1qQXdOakl3V2pBQU1Ga3dFd1lICktvWkl6ajBDQVFZSUtvWkl6ajBEQVFjRFFnQUVmc2Z5NXg1cEd5TUszelZiVVdBUTg2bFU2d29mVmdWVC9sSXgKZHlvbTVZT1FBam9WZG1uMkV3SzR1L1drTXlSUTBZeXZHVnNnQkpmNERYZ0tRUVNSZktPQ0FaRXdnZ0dOTUE0RwpBMVVkRHdFQi93UUVBd0lIZ0RBVEJnTlZIU1VFRERBS0JnZ3JCZ0VGQlFjREF6QWRCZ05WSFE0RUZnUVVGdnZaCllIay9abWFnOWZINlRjcTNYRTZieXdzd0h3WURWUjBqQkJnd0ZvQVUzOVBwejFZa0VaYjVxTmpwS0ZXaXhpNFkKWkQ4d1FRWURWUjBSQVFIL0JEY3dOWUV6WTI5emFXZHVRR052YzJsbmJpMTBaWE4wTFd0aGJtbHJieTB4TG1saApiUzVuYzJWeWRtbGpaV0ZqWTI5MWJuUXVZMjl0TUNrR0Npc0dBUVFCZzc4d0FRRUVHMmgwZEhCek9pOHZZV05qCmIzVnVkSE11WjI5dloyeGxMbU52YlRBckJnb3JCZ0VFQVlPL01BRUlCQjBNRzJoMGRIQnpPaTh2WVdOamIzVnUKZEhNdVoyOXZaMnhsTG1OdmJUQ0JpZ1lLS3dZQkJBSFdlUUlFQWdSOEJIb0FlQUIyQU4wOU1Hckd4eEV5WXhrZQpISmxuTndLaVNsNjQzanl0LzRlS2NvQXZLZTZPQUFBQmg2cUo4NFFBQUFRREFFY3dSUUlnT2EvY0hQeXlldXFUCmQxaEl1U3RzL2RxUkhpTXUxZ1lKZTNobHlOS3VRbU1DSVFDQWd0Z1hoTCtlZkdoSDVLY1VNU1FQYmxJc2lNTjMKbk9hTTBmemN0Wk9MdWpBS0JnZ3Foa2pPUFFRREF3Tm9BREJsQWpFQXZ2SlZoZGtpRDNPL2gyVEVua0dQcXJ4dAp0NzlrZG4weWdRSjFiS2xLS3ZRRnVPRzJIZnBDa25TcXJDYm1QMVlEQWpCVkcyeTFub3NzZkh6ak9RNE85SS92CnRER0hpenRqNG0xbmV0dGJjT2dPdEhUejl3SVFaSTA4QWovUG9nU0IyVzA9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K"
      }
    }
  }
}
```

attestation with sbom.  Note the kaniko attestation here includes the full cyclonedx output for **ALL** packages (including go packages)

```bash
rekor-cli get --rekor_server https://rekor.sigstore.dev \
   --uuid 24296fb24b8ad77a61fa48c55a19124687ede29f830a6478d0316128ed4785cefcbde5d1423275e8         

LogID: c0d23d6ad406973f9559f3ba2d1ca01f84147d8ffc5b8445c224f98b9591801d
Attestation: {"_type":"https://in-toto.io/Statement/v0.1","predicateType":"https://cyclonedx.org/bom/v1.4","subject":[{"name":"us-central1-docker.pkg.dev/cosign-test-kaniko-1-384813/repo1/securebuild","digest":{"sha256":"5e065a75f1dd137db2a1d7a5aada2160668cb5cb705732d8877e2c362cfc5935"}}],"predicate":{"bomFormat":"CycloneDX","components":[{"bom-ref":"pkg:deb/debian/base-files@10.3+deb10u9?arch=amd64\u0026distro=debian-10\u0026package-id=5aa6e4929bf16696","cpe":"cpe:2.3:a:base-files:base-files:10.3\\+deb10u9:*:*:*:*:*:*:*","licenses":[{"license":{"name":"GPL"}}],"name":"base-files","properties":[{"name":"syft:package:foundBy","value":"dpkgdb-cataloger"},{"name":"syft:package:metadataType","value":"DpkgMetadata"},{"name":"syft:package:type","value":"deb"},{"name":"syft:cpe23","value":"cpe:2.3:a:base-files:base_files:10.3\\+deb10u9:*:*:*:*:*:*:*"},{"name":"syft:cpe23","value":"cpe:2.3:a:base_files:base-files:10.3\\+deb10u9:*:*:*:*:*:*:*"},{"name":"syft:cpe23","value":"cpe:2.3:a:base_files:base_files:10.3\\+deb10u9:*:*:*:*:*:*:*"},{"name":"syft:cpe23","value":"cpe:2.3:a:base:base-files:10.3\\+deb10u9:*:*:*:*:*:*:*"},{"name":"syft:cpe23","value":"cpe:2.3:a:base:base_files:10.3\\+deb10u9:*:*:*:*:*:*:*"},{"name":"syft:location:0:layerID","value":"sha256:b5c218e3bb6075af8dcb55250a52f873d3469437f465cf5d8e852e0421c085b8"},{"name":"syft:location:0:path","value":"/usr/share/doc/base-files/copyright"},{"name":"syft:location:1:layerID","value":"sha256:b5c218e3bb6075af8dcb55250a52f873d3469437f465cf5d8e852e0421c085b8"},{"name":"syft:location:1:path","value":"/var/lib/dpkg/status.d/base"},{"name":"syft:metadata:installedSize","value":"340"}],"publisher":"Santiago Vila \u003csanvila@debian.org\u003e","purl":"pkg:deb/debian/base-files@10.3+deb10u9?arch=amd64\u0026distro=debian-10","type":"library","version":"10.3+deb10u9"},{"bom-ref":"pkg:golang/github.com/gorilla/mux@v1.8.0?package-id=b8fb11cf7e63c7fc","cpe":"cpe:2.3:a:gorilla:mux:v1.8.0:*:*:*:*:*:*:*","name":"github.com/gorilla/mux","properties":[{"name":"syft:package:foundBy","value":"go-module-binary-cataloger"},{"name":"syft:package:language","value":"go"},{"name":"syft:package:metadataType","value":"GolangBinMetadata"},{"name":"syft:package:type","value":"go-module"},{"name":"syft:location:0:layerID","value":"sha256:e6e79b86459a23fd3a9632f825daaea3b6b826767fc716153fe2823d53ec6a54"},{"name":"syft:location:0:path","value":"/server"},{"name":"syft:metadata:architecture","value":"amd64"},{"name":"syft:metadata:goCompiledVersion","value":"go1.19.8"},{"name":"syft:metadata:h1Digest","value":"h1:i40aqfkR1h2SlN9hojwV5ZA91wcXFOvkdNIeFDP5koI="},{"name":"syft:metadata:mainModule","value":"github.com/salrashid123/cosign_bazel_cloud_build/app"}],"purl":"pkg:golang/github.com/gorilla/mux@v1.8.0","type":"library","version":"v1.8.0"},{"bom-ref":"pkg:golang/github.com/salrashid123/cosign_bazel_cloud_build/app@(devel)?package-id=a4b8b8266ee720f1","cpe":"cpe:2.3:a:salrashid123:cosign-bazel-cloud-build\\/app:\\(devel\\):*:*:*:*:*:*:*","name":"github.com/salrashid123/cosign_bazel_cloud_build/app","properties":[{"name":"syft:package:foundBy","value":"go-module-binary-cataloger"},{"name":"syft:package:language","value":"go"},{"name":"syft:package:metadataType","value":"GolangBinMetadata"},{"name":"syft:package:type","value":"go-module"},{"name":"syft:cpe23","value":"cpe:2.3:a:salrashid123:cosign_bazel_cloud_build\\/app:\\(devel\\):*:*:*:*:*:*:*"},{"name":"syft:location:0:layerID","value":"sha256:e6e79b86459a23fd3a9632f825daaea3b6b826767fc716153fe2823d53ec6a54"},{"name":"syft:location:0:path","value":"/server"},{"name":"syft:metadata:architecture","value":"amd64"},{"name":"syft:metadata:goBuildSettings:-compiler","value":"gc"},{"name":"syft:metadata:goBuildSettings:CGO_ENABLED","value":"1"},{"name":"syft:metadata:goBuildSettings:GOAMD64","value":"v1"},{"name":"syft:metadata:goBuildSettings:GOARCH","value":"amd64"},{"name":"syft:metadata:goBuildSettings:GOOS","value":"linux"},{"name":"syft:metadata:goCompiledVersion","value":"go1.19.8"},{"name":"syft:metadata:mainModule","value":"github.com/salrashid123/cosign_bazel_cloud_build/app"}],"purl":"pkg:golang/github.com/salrashid123/cosign_bazel_cloud_build/app@(devel)","type":"library","version":"(devel)"},{"bom-ref":"pkg:golang/golang.org/x/net@v0.0.0-20220921203646-d300de134e69?package-id=54a64e800919b8c7","cpe":"cpe:2.3:a:golang:x\\/net:v0.0.0-20220921203646-d300de134e69:*:*:*:*:*:*:*","name":"golang.org/x/net","properties":[{"name":"syft:package:foundBy","value":"go-module-binary-cataloger"},{"name":"syft:package:language","value":"go"},{"name":"syft:package:metadataType","value":"GolangBinMetadata"},{"name":"syft:package:type","value":"go-module"},{"name":"syft:location:0:layerID","value":"sha256:e6e79b86459a23fd3a9632f825daaea3b6b826767fc716153fe2823d53ec6a54"},{"name":"syft:location:0:path","value":"/server"},{"name":"syft:metadata:architecture","value":"amd64"},{"name":"syft:metadata:goCompiledVersion","value":"go1.19.8"},{"name":"syft:metadata:h1Digest","value":"h1:hUJpGDpnfwdJW8iNypFjmSY0sCBEL+spFTZ2eO+Sfps="},{"name":"syft:metadata:mainModule","value":"github.com/salrashid123/cosign_bazel_cloud_build/app"}],"purl":"pkg:golang/golang.org/x/net@v0.0.0-20220921203646-d300de134e69","type":"library","version":"v0.0.0-20220921203646-d300de134e69"},{"bom-ref":"pkg:golang/golang.org/x/text@v0.3.7?package-id=4df8f317ccc61a57","cpe":"cpe:2.3:a:golang:x\\/text:v0.3.7:*:*:*:*:*:*:*","name":"golang.org/x/text","properties":[{"name":"syft:package:foundBy","value":"go-module-binary-cataloger"},{"name":"syft:package:language","value":"go"},{"name":"syft:package:metadataType","value":"GolangBinMetadata"},{"name":"syft:package:type","value":"go-module"},{"name":"syft:location:0:layerID","value":"sha256:e6e79b86459a23fd3a9632f825daaea3b6b826767fc716153fe2823d53ec6a54"},{"name":"syft:location:0:path","value":"/server"},{"name":"syft:metadata:architecture","value":"amd64"},{"name":"syft:metadata:goCompiledVersion","value":"go1.19.8"},{"name":"syft:metadata:h1Digest","value":"h1:olpwvP2KacW1ZWvsR7uQhoyTYvKAupfQrRGBFM352Gk="},{"name":"syft:metadata:mainModule","value":"github.com/salrashid123/cosign_bazel_cloud_build/app"}],"purl":"pkg:golang/golang.org/x/text@v0.3.7","type":"library","version":"v0.3.7"},{"bom-ref":"pkg:deb/debian/libc6@2.28-10?arch=amd64\u0026upstream=glibc\u0026distro=debian-10\u0026package-id=74ac5ee7adfb6a2d","cpe":"cpe:2.3:a:libc6:libc6:2.28-10:*:*:*:*:*:*:*","licenses":[{"license":{"id":"GPL-2.0-only"}},{"license":{"id":"LGPL-2.1-only"}}],"name":"libc6","properties":[{"name":"syft:package:foundBy","value":"dpkgdb-cataloger"},{"name":"syft:package:metadataType","value":"DpkgMetadata"},{"name":"syft:package:type","value":"deb"},{"name":"syft:location:0:layerID","value":"sha256:f4d542ed5244573730cc1531492c544507f8b1e7eabda02abf10ca7b5e937589"},{"name":"syft:location:0:path","value":"/usr/share/doc/libc6/copyright"},{"name":"syft:location:1:layerID","value":"sha256:f4d542ed5244573730cc1531492c544507f8b1e7eabda02abf10ca7b5e937589"},{"name":"syft:location:1:path","value":"/var/lib/dpkg/status.d/libc6"},{"name":"syft:metadata:installedSize","value":"12337"},{"name":"syft:metadata:source","value":"glibc"}],"publisher":"GNU Libc Maintainers \u003cdebian-glibc@lists.debian.org\u003e","purl":"pkg:deb/debian/libc6@2.28-10?arch=amd64\u0026upstream=glibc\u0026distro=debian-10","type":"library","version":"2.28-10"},{"bom-ref":"pkg:deb/debian/libssl1.1@1.1.1d-0+deb10u6?arch=amd64\u0026upstream=openssl\u0026distro=debian-10\u0026package-id=ab8b40f4f3d74be0","cpe":"cpe:2.3:a:libssl1.1:libssl1.1:1.1.1d-0\\+deb10u6:*:*:*:*:*:*:*","name":"libssl1.1","properties":[{"name":"syft:package:foundBy","value":"dpkgdb-cataloger"},{"name":"syft:package:metadataType","value":"DpkgMetadata"},{"name":"syft:package:type","value":"deb"},{"name":"syft:location:0:layerID","value":"sha256:f4d542ed5244573730cc1531492c544507f8b1e7eabda02abf10ca7b5e937589"},{"name":"syft:location:0:path","value":"/usr/share/doc/libssl1.1/copyright"},{"name":"syft:location:1:layerID","value":"sha256:f4d542ed5244573730cc1531492c544507f8b1e7eabda02abf10ca7b5e937589"},{"name":"syft:location:1:path","value":"/var/lib/dpkg/status.d/libssl1"},{"name":"syft:metadata:installedSize","value":"4077"},{"name":"syft:metadata:source","value":"openssl"}],"publisher":"Debian OpenSSL Team \u003cpkg-openssl-devel@lists.alioth.debian.org\u003e","purl":"pkg:deb/debian/libssl1.1@1.1.1d-0+deb10u6?arch=amd64\u0026upstream=openssl\u0026distro=debian-10","type":"library","version":"1.1.1d-0+deb10u6"},{"bom-ref":"pkg:deb/debian/netbase@5.6?arch=all\u0026distro=debian-10\u0026package-id=b55e51dca4eba9a6","cpe":"cpe:2.3:a:netbase:netbase:5.6:*:*:*:*:*:*:*","licenses":[{"license":{"id":"GPL-2.0-only"}}],"name":"netbase","properties":[{"name":"syft:package:foundBy","value":"dpkgdb-cataloger"},{"name":"syft:package:metadataType","value":"DpkgMetadata"},{"name":"syft:package:type","value":"deb"},{"name":"syft:location:0:layerID","value":"sha256:b5c218e3bb6075af8dcb55250a52f873d3469437f465cf5d8e852e0421c085b8"},{"name":"syft:location:0:path","value":"/usr/share/doc/netbase/copyright"},{"name":"syft:location:1:layerID","value":"sha256:b5c218e3bb6075af8dcb55250a52f873d3469437f465cf5d8e852e0421c085b8"},{"name":"syft:location:1:path","value":"/var/lib/dpkg/status.d/netbase"},{"name":"syft:metadata:installedSize","value":"44"}],"publisher":"Marco d'Itri \u003cmd@linux.it\u003e","purl":"pkg:deb/debian/netbase@5.6?arch=all\u0026distro=debian-10","type":"library","version":"5.6"},{"bom-ref":"pkg:deb/debian/openssl@1.1.1d-0+deb10u6?arch=amd64\u0026distro=debian-10\u0026package-id=5baa662d4c747c2e","cpe":"cpe:2.3:a:openssl:openssl:1.1.1d-0\\+deb10u6:*:*:*:*:*:*:*","name":"openssl","properties":[{"name":"syft:package:foundBy","value":"dpkgdb-cataloger"},{"name":"syft:package:metadataType","value":"DpkgMetadata"},{"name":"syft:package:type","value":"deb"},{"name":"syft:location:0:layerID","value":"sha256:f4d542ed5244573730cc1531492c544507f8b1e7eabda02abf10ca7b5e937589"},{"name":"syft:location:0:path","value":"/usr/share/doc/openssl/copyright"},{"name":"syft:location:1:layerID","value":"sha256:f4d542ed5244573730cc1531492c544507f8b1e7eabda02abf10ca7b5e937589"},{"name":"syft:location:1:path","value":"/var/lib/dpkg/status.d/openssl"},{"name":"syft:metadata:installedSize","value":"1460"}],"publisher":"Debian OpenSSL Team \u003cpkg-openssl-devel@lists.alioth.debian.org\u003e","purl":"pkg:deb/debian/openssl@1.1.1d-0+deb10u6?arch=amd64\u0026distro=debian-10","type":"library","version":"1.1.1d-0+deb10u6"},{"bom-ref":"pkg:deb/debian/tzdata@2021a-0+deb10u1?arch=all\u0026distro=debian-10\u0026package-id=9e5b2198bbbd7fb0","cpe":"cpe:2.3:a:tzdata:tzdata:2021a-0\\+deb10u1:*:*:*:*:*:*:*","name":"tzdata","properties":[{"name":"syft:package:foundBy","value":"dpkgdb-cataloger"},{"name":"syft:package:metadataType","value":"DpkgMetadata"},{"name":"syft:package:type","value":"deb"},{"name":"syft:location:0:layerID","value":"sha256:b5c218e3bb6075af8dcb55250a52f873d3469437f465cf5d8e852e0421c085b8"},{"name":"syft:location:0:path","value":"/usr/share/doc/tzdata/copyright"},{"name":"syft:location:1:layerID","value":"sha256:b5c218e3bb6075af8dcb55250a52f873d3469437f465cf5d8e852e0421c085b8"},{"name":"syft:location:1:path","value":"/var/lib/dpkg/status.d/tzdata"},{"name":"syft:metadata:installedSize","value":"3040"}],"publisher":"GNU Libc Maintainers \u003cdebian-glibc@lists.debian.org\u003e","purl":"pkg:deb/debian/tzdata@2021a-0+deb10u1?arch=all\u0026distro=debian-10","type":"library","version":"2021a-0+deb10u1"},{"description":"Distroless","externalReferences":[{"type":"issue-tracker","url":"https://github.com/GoogleContainerTools/distroless/issues/new"},{"type":"website","url":"https://github.com/GoogleContainerTools/distroless"},{"comment":"support","type":"other","url":"https://github.com/GoogleContainerTools/distroless/blob/master/README.md"}],"name":"debian","properties":[{"name":"syft:distro:id","value":"debian"},{"name":"syft:distro:prettyName","value":"Distroless"},{"name":"syft:distro:versionID","value":"10"}],"swid":{"name":"debian","tagId":"debian","version":"10"},"type":"operating-system","version":"10"}],"metadata":{"component":{"bom-ref":"b8a4c1139c17fd58","name":"us-central1-docker.pkg.dev/cosign-test-kaniko-1-384813/repo1/securebuild@sha256:5e065a75f1dd137db2a1d7a5aada2160668cb5cb705732d8877e2c362cfc5935","type":"container","version":"sha256:8695e08ead32b1027e407da433049e8b23a357b13cdf20a825b06f0352d24c00"},"timestamp":"2023-04-25T13:11:03Z","tools":[{"name":"syft","vendor":"anchore","version":"0.78.0"}]},"serialNumber":"urn:uuid:0cdddce1-620f-4ab1-b2c6-968db5e02b9c","specVersion":"1.4","version":1}}
Index: 18896193
IntegratedTime: 2023-04-25T13:11:05Z
UUID: 24296fb24b8ad77a61fa48c55a19124687ede29f830a6478d0316128ed4785cefcbde5d1423275e8
Body: {
  "IntotoObj": {
    "content": {
      "hash": {
        "algorithm": "sha256",
        "value": "7ee142fc8c2d08b29fcb53fa5e1b9af5a3d65f8a792ae9503e257c0b9cab6f97"
      },
      "payloadHash": {
        "algorithm": "sha256",
        "value": "b350df4d65b154a3538b74b27ef8c4a561be12d0bce2cee30c4867abe23e9854"
      }
    },
    "publicKey": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM5RENDQW5tZ0F3SUJBZ0lVSlFNQ0dWeWZLeGFMaGw4ZWUyMzNzaytxZlgwd0NnWUlLb1pJemowRUF3TXcKTnpFVk1CTUdBMVVFQ2hNTWMybG5jM1J2Y21VdVpHVjJNUjR3SEFZRFZRUURFeFZ6YVdkemRHOXlaUzFwYm5SbApjbTFsWkdsaGRHVXdIaGNOTWpNd05ESTFNVE14TVRBMFdoY05Nak13TkRJMU1UTXlNVEEwV2pBQU1Ga3dFd1lICktvWkl6ajBDQVFZSUtvWkl6ajBEQVFjRFFnQUU4RzFXS0s5S0FiVWZ2eFBJRnIyUkpSMmsxcUl4Z3kzM0dWQjAKb2hiWW1mNWJXWU5NSExWa3hZVTFBWk9DQkhxTGRNOEMycGdLTzZxYllOVE9CenBRUTZPQ0FaZ3dnZ0dVTUE0RwpBMVVkRHdFQi93UUVBd0lIZ0RBVEJnTlZIU1VFRERBS0JnZ3JCZ0VGQlFjREF6QWRCZ05WSFE0RUZnUVVsRUhHCkRydkRQeTZObzFxSzRUdVpoMmN1OGtzd0h3WURWUjBqQkJnd0ZvQVUzOVBwejFZa0VaYjVxTmpwS0ZXaXhpNFkKWkQ4d1NBWURWUjBSQVFIL0JENHdQSUU2WTI5emFXZHVRR052YzJsbmJpMTBaWE4wTFd0aGJtbHJieTB4TFRNNApORGd4TXk1cFlXMHVaM05sY25acFkyVmhZMk52ZFc1MExtTnZiVEFwQmdvckJnRUVBWU8vTUFFQkJCdG9kSFJ3CmN6b3ZMMkZqWTI5MWJuUnpMbWR2YjJkc1pTNWpiMjB3S3dZS0t3WUJCQUdEdnpBQkNBUWREQnRvZEhSd2N6b3YKTDJGalkyOTFiblJ6TG1kdmIyZHNaUzVqYjIwd2dZb0dDaXNHQVFRQjFua0NCQUlFZkFSNkFIZ0FkZ0RkUFRCcQp4c2NSTW1NWkhoeVpaemNDb2twZXVONDhyZitIaW5LQUx5bnVqZ0FBQVllNGlnRE5BQUFFQXdCSE1FVUNJUUMvCjQ1SzhHeDllSHA0RHN4YnlheFVnbXpOUDdrd0o5ZEFLd3h4S0ZrZklUZ0lnQks0QTlsT1VDRmJPVjUzNjZhdHMKNm1OSFpZRmNUNnJpYmRwbExQS1FGQUV3Q2dZSUtvWkl6ajBFQXdNRGFRQXdaZ0l4QUx2dDJnMnBQSmdwRHlaUwp5RHpFemR1a21UUlM0azhRS1ZhdVVpTStXLzc3eDJMVUQ3dmhZYU9tNUt0MnE1Z0NOUUl4QUtRUTJ6MldWdytxCmcrc05YVnhNOXcrSFE2aGJ2cWR2dGUrckhmd1NqaXFDcDYzZm8rb01GMytHNDk2NThxTzNNQT09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K"
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
            25:03:02:19:5c:9f:2b:16:8b:86:5f:1e:7b:6d:f7:b2:4f:aa:7d:7d
        Signature Algorithm: ecdsa-with-SHA384
        Issuer: O = sigstore.dev, CN = sigstore-intermediate
        Validity
            Not Before: Apr 25 13:11:04 2023 GMT
            Not After : Apr 25 13:21:04 2023 GMT
        Subject: 
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:f0:6d:56:28:af:4a:01:b5:1f:bf:13:c8:16:bd:
                    91:25:1d:a4:d6:a2:31:83:2d:f7:19:50:74:a2:16:
                    d8:99:fe:5b:59:83:4c:1c:b5:64:c5:85:35:01:93:
                    82:04:7a:8b:74:cf:02:da:98:0a:3b:aa:9b:60:d4:
                    ce:07:3a:50:43
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature
            X509v3 Extended Key Usage: 
                Code Signing
            X509v3 Subject Key Identifier: 
                94:41:C6:0E:BB:C3:3F:2E:8D:A3:5A:8A:E1:3B:99:87:67:2E:F2:4B
            X509v3 Authority Key Identifier: 
                DF:D3:E9:CF:56:24:11:96:F9:A8:D8:E9:28:55:A2:C6:2E:18:64:3F
            X509v3 Subject Alternative Name: critical
                email:cosign@cosign-test-kaniko-1-384813.iam.gserviceaccount.com          <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
            1.3.6.1.4.1.57264.1.1: 
                https://accounts.google.com
            1.3.6.1.4.1.57264.1.8: 
                ..https://accounts.google.com
            CT Precertificate SCTs: 
                Signed Certificate Timestamp:
                    Version   : v1 (0x0)
                    Log ID    : DD:3D:30:6A:C6:C7:11:32:63:19:1E:1C:99:67:37:02:
                                A2:4A:5E:B8:DE:3C:AD:FF:87:8A:72:80:2F:29:EE:8E
                    Timestamp : Apr 25 13:11:04.653 2023 GMT
                    Extensions: none
                    Signature : ecdsa-with-SHA256
                                30:45:02:21:00:BF:E3:92:BC:1B:1F:5E:1E:9E:03:B3:
                                16:F2:6B:15:20:9B:33:4F:EE:4C:09:F5:D0:0A:C3:1C:
                                4A:16:47:C8:4E:02:20:04:AE:00:F6:53:94:08:56:CE:
                                57:9D:FA:E9:AB:6C:EA:63:47:65:81:5C:4F:AA:E2:6D:
                                DA:65:2C:F2:90:14:01
    Signature Algorithm: ecdsa-with-SHA384
    Signature Value:
        30:66:02:31:00:bb:ed:da:0d:a9:3c:98:29:0f:26:52:c8:3c:
        c4:cd:db:a4:99:34:52:e2:4f:10:29:56:ae:52:23:3e:5b:fe:
        fb:c7:62:d4:0f:bb:e1:61:a3:a6:e4:ab:76:ab:98:02:35:02:
        31:00:a4:10:db:3d:96:57:0f:aa:83:eb:0d:5d:5c:4c:f7:0f:
        87:43:a8:5b:be:a7:6f:b5:ef:ab:1d:fc:12:8e:2a:82:a7:ad:
        df:a3:ea:0c:17:7f:86:e3:de:b9:f2:a3:b7:30
```