# helloworld-container-signing

Introduction, practice and tutorial for achieving
supply chain security in the container space.

## Docker container images example projects

Build the provided `Dockerfile` as a 
local container image:

```sh
docker build -t hello-container-signing .
```

Then run it:

```sh
docker run --rm hello-container-signing
```

## Installing the Cosign command line tool

To get started with container signing we'll need a copy
of the `cosign` command-line tool so we can:
1. Generate our own maintainer key-pair
2. Verify container images

If you're on macOS, install `cosign` as follows for the
easiest and fastest way:

```sh
brew install cosign
```

For other installation targets you may head over to [releases](https://github.com/sigstore/cosign/releases/latest)
in order to find the one relevant to your CPU architecture
and particular need.

## Generate public and private key-pair signature with Cosign

The `cosign` tool accepts several methods of passing
the private password in order to generate a key-pair
type of key signature, such as:
1. Pass it as STDIN: `echo "1234" | cosign generate-key-pair`
2. Type into an interactive standard interface: `cosign generate-key-pair`
3. Use an environment variable `COSIGN_PASSWORD=1234 cosign generate-key-pair`

You may use whichever you prefer, and should output similar
to the following:

```sh
Enter password for private key:
Enter again:
Private key written to cosign.key
Public key written to cosign.pub
```

Once the key-pair has been successfully generated, it will
result in the following files created on disk, in the current
working directory:
1. `cosign.key` which is the private signing key
2. `cosign.pub` which is the public signing key

## Add key-pair signature to GitHub Actions for CI signing 

To allow for automated signing of container images built
and published to a repository, we need to make the key-pair
available as environment secrets on the GitHub Actions CI.

Head over to GitHub's repository settings at https://github.com/lirantal/dockly/settings/secrets/actions/new
and created the following environment variables, with their
respective values from the key-pair files that were generated
earlier:
1. `COSIGN_PUBLIC_KEY` - public key from the generated file `cosign.pub`
2. `COSIGN_PRIVATE_KEY` - private key from the generated file `cosign.key`
3. `COSIGN_PASSWORD` - the value of the password provided to `cosign generate-key-pair`
   earlier 

Now, that these set of sensitive information have been set on
the repository it is advised that you remove the private key
`cosign.key` from the filesystem, so it isn't leaked by an
accidental commit to a public repository, mistakenly added to
a built container image, or some other reason for credential leak.

Furthermore, it is advised you keep a copy of the above sensitive
information in the likes of a key management system, such as 1Password.
You'll need access to all of them, including the public key.

## Sign a container image with Cosign and GitHub Actions

Add the following to your GitHub Actions workflow, which handles
the installation of the Cosign utility as part of the workflow,
and then continues to sign the image:

```yaml
      - name: Cosign install
        uses: sigstore/cosign-installer@v2.4.0
              
      - name: Sign the published container image
        env:
          COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
          COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
          TAGS: ${{ steps.meta.outputs.tags }}
        run: cosign sign --key env://COSIGN_PRIVATE_KEY ${TAGS}
```

And here's a reference to a complete container image build and
publishing to the GitHub Packages container registry for the
open source project `dockly` which serves as an example reference:

```yaml
name: "Docker: GitHub Packages"

# This workflow uses actions that are not certified by GitHub.
# They are provided by a third-party and are governed by
# separate terms of service, privacy policy, and support
# documentation.

on:
  push:
    branches: [ main ]
    tags: [ 'v*.*.*' ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository_owner }}/dockly

jobs:
  build_and_publish:
  
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Log into registry ${{ env.REGISTRY }}
        if: github.event_name != 'pull_request'
        uses: docker/login-action@28218f9b04b4f3f62068d7b6ce6ca5b26e35336c
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.repository_owner }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@98669ae865ea3cffbcbaa878cf57c20bbf1c6c38
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          flavor: |
            latest=true
            prefix=
            suffix=

      - name: Build and push Docker image
        uses: docker/build-push-action@ad44023a93711e3deb337508980b4b5e9bcdc5dc
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          
      - name: Cosign install
        uses: sigstore/cosign-installer@v2.4.0
              
      - name: Sign the published container image
        env:
          COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
          COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
          TAGS: ${{ steps.meta.outputs.tags }}
        run: cosign sign --key env://COSIGN_PRIVATE_KEY ${TAGS}
```

## Verifying a container image signature with Cosign

Once the GitHub Action workflow completes successfully in building
and signing the image at the GitHub Packages container registry
we can proceed to verifying the container image signature.

This step of verifying container image signatures is mostly reserved
and to be used by end-users who consume the container images.
Verifying Docker images or other OCI-compliant container images
is performed in order to ensure that specific claims have been
correctly signed and that overall provenance tested for integrity
is correct 

To verify the image, we will need access to the public key of the
maintainer, which was used to sign the image. Since we have created
this one in prior steps, it should be easily available on disk:

```sh
cosign verify ghcr.io/lirantal/dockly --key cosign.pub
```

This should output similar information to the following:

```sh
Verification for ghcr.io/lirantal/dockly:latest --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key

[{"critical":{"identity":{"docker-reference":"ghcr.io/lirantal/dockly"},"image":{"docker-manifest-digest":"sha256:3cb3a861c4077c63a48a523c1ab74c0e6e737fae112b8f8602718509385a74fd"},"type":"cosign container image signature"},"optional":null}]
```

Tip: you can pipe the output of `cosign verify` to the popular JSON
querying tool `jq` and get a pretty-print version such as this:

```sh
cosign verify ghcr.io/lirantal/dockly --key cosign.pub  | jq

Verification for ghcr.io/lirantal/dockly:latest --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key
[
  {
    "critical": {
      "identity": {
        "docker-reference": "ghcr.io/lirantal/dockly"
      },
      "image": {
        "docker-manifest-digest": "sha256:3cb3a861c4077c63a48a523c1ab74c0e6e737fae112b8f8602718509385a74fd"
      },
      "type": "cosign container image signature"
    },
    "optional": null
  }
]
```

## Verify a container image provenance

So far, we have confirmed that this specific container image
tag was signed with the public key that we used. This is one 
attestation with regards to a claim about the identity who
published this Docker image.

What if we also want to track further attestations, such as
the source repository of this container image, and the specific
build workflow origin and commit reference it is related to?

You can look up the relevant information you wish to include
as attestations for the overall container image provenance in
[GitHub's Actions documentation about Contexts](https://docs.github.com/en/actions/learn-github-actions/contexts).

I have updated the signing clause of the workflow file with
the following claims to be added to the signature:

```yaml
      - name: Sign the published container image
        env:
          COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
          COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
          TAGS: ${{ steps.meta.outputs.tags }}
        run: |
          cosign sign --key env://COSIGN_PRIVATE_KEY ${TAGS} \
            -a "repo=${{ github.repository }}" \
            -a "workflow=${{ github.workflow }}" \
            -a "ref=${{ github.sha }}" \
            -a "actor=${{ github.actor }}" \
            -a "build=${{ github.run_id }}"
```

Once the build completed successfully we can verify the container
image along with the new attestations added to it in the `optional`
field:

```sh
$ cosign verify ghcr.io/lirantal/dockly --key cosign.pub  | jq

Verification for ghcr.io/lirantal/dockly:latest --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key
[
  {
    "critical": {
      "identity": {
        "docker-reference": "ghcr.io/lirantal/dockly"
      },
      "image": {
        "docker-manifest-digest": "sha256:2bbff3db48fcae016aa2d2116834723591a40335f47be6df81c532170b45776b"
      },
      "type": "cosign container image signature"
    },
    "optional": {
      "actor": "lirantal",
      "build": "2535552796",
      "ref": "a40c60c8dc6d8d970431606b87df01dc71486b6c",
      "repo": "lirantal/dockly",
      "workflow": "Docker: GitHub Packages"
    }
  }
]
```

