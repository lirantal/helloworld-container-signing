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

