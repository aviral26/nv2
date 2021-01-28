# Push, Discover, Pull Prototype

Available here: https://github.com/aviral26/distribution/tree/prototype-2

The following steps illustrate how signatures can be stored and retrieved from a registry.

### Prerequisites

- Local registry prototype instance
- [docker-generate](https://github.com/shizhMSFT/docker-generate)
- [nv2](https://github.com/notaryproject/nv2)
- `curl`
- `jq`

### Push an image to your registry

```shell
# Local registry
regIp="127.0.0.1" && \
  regPort="5000" && \
  registry="$regIp:$regPort" && \
  repo="busybox" && \
  tag="latest" && \
  image="$repo:$tag" && \
  reference="$registry/$image"

# Pull image from docker hub and push to local registry
docker pull $image && \
  docker tag $image $reference && \
  docker push $reference
```

### Generate image manifest and sign it

```shell
# Generate self-signed certificates
openssl req \
  -x509 \
  -sha256 \
  -nodes \
  -newkey rsa:2048 \
  -days 365 \
  -subj "/CN=$regIp/O=example inc/C=IN/ST=Haryana/L=Gurgaon" \
  -addext "subjectAltName=IP:$regIp" \
  -keyout example.key \
  -out example.crt

# Generate image manifest
manifestFile="manifest-to-sign.json" && \
  docker generate manifest $image > $manifestFile

# Sign manifest
signatureFile="manifest-signature.jwt" && \
  nv2 sign --method x509 \
    -k example.key \
    -c example.crt \
    -r $reference \
    -o $signatureFile \
    file:$manifestFile
```

### Obtain manifest and signature digests

```shell
manifestDigest="sha256:$(sha256sum $manifestFile | cut -d " " -f 1)" && \
  signatureDigest="sha256:$(sha256sum $signatureFile | cut -d " " -f 1)"
```

### Create an Artifact file referencing the manifest that was signed and its signature as config

```shell
artifactFile="artifact.json" && \
  artifactMediaType="application/vnd.oci.artifact.manifest.v1+json" && \
  artifactType="application/vnd.cncf.notary.v2" && \
  configMediaType="application/vnd.cncf.notary.config.v2+jwt" && \
  signatureFileSize=`wc -c < $signatureFile` && \
  manifestMediaType="$(cat $manifestFile | jq -r '.mediaType')" && \
  manifestFileSize=`wc -c < $manifestFile`

cat <<EOF > $artifactFile
{
  "schemaVersion": 2,
  "mediaType": "$artifactMediaType",
  "artifactType": "$artifactType",
  "config": {
    "mediaType": "$configMediaType",
    "digest": "$signatureDigest",
    "size": $signatureFileSize
  },
  "manifests": [
    {
      "mediaType": "$manifestMediaType",
      "digest": "$manifestDigest",
      "size": $manifestFileSize
    }
  ]
}
EOF
```

### Obtain artifact digest

```shell
artifactDigest="sha256:$(sha256sum $artifactFile | cut -d " " -f 1)"
```

### Push signature and artifact

```shell
# Initiate blob upload and obtain PUT location
configPutLocation=`curl -I -X POST -s http://$registry/v2/$repo/blobs/uploads/ | grep "Location: " | sed -e "s/Location: //;s/$/\&digest=$signatureDigest/;s/\r//"`

# Push signature blob
curl -X PUT -H "Content-Type: application/octet-stream" --data-binary @"$signatureFile" $configPutLocation

# Push artifact
curl -X PUT --data-binary @"$artifactFile" -H "Content-Type: $artifactMediaType" "http://$registry/v2/$repo/manifests/$artifactDigest"
```

### Retrieve signatures of a manifest as referrer metadata

```shell
# Retrieve linked artifacts
curl -s "http://$registry/v2/_ext/oci-artifacts/v1/$repo/manifests/$manifestDigest/links?artifact-type=$artifactType" | jq
```

### Verify signature

```shell
# Retrieve signature digest from first artifact.
signatureDigest=`curl -s "http://$registry/v2/_ext/oci-artifacts/v1/$repo/manifests/$manifestDigest/links?artifact-type=$artifactType" | jq -r '.links[0].config.digest'` && \
  retrievedSignatureFile="retrieved-signature.json" && \
  curl -s http://$registry/v2/$repo/blobs/$signatureDigest > $retrievedSignatureFile

# Verify signature
nv2 verify \
  -f $retrievedSignatureFile \
  -c example.crt \
  file:$manifestFile
```
