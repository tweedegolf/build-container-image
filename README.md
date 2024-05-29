# Creating a container image
You can use this workflow step to create a new container image, this mostly
relies on the docker/build-push-action, but sets up the environment for easy
mutli-arch builds with buildx and qemu. It also logs in to the GitHub container
registry allowing you to upload an image right away. To use it, add a job to
your GitHub workflow calling this workflow:

```yaml
# ...

jobs:

  # ...

  docker:
    runs-on: ubuntu-latest
    steps:
      # ...

      - name: Build container image
        uses: tweedegolf/build-container-image@main
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          push: ${{ github.ref == 'refs/heads/main' }}
          platforms: "linux/amd64,linux/arm64"
          tags: ghcr.io/tweedegolf/example:latest

      # ...

  # ...
```

### Workflow inputs
This workflow has several parameters that allow customizing the behavior:

<table>
  <thead>
    <tr><th>Option</th><th>Required</th><th>Default</th></tr>
  </thead>
  <tbody>
    <tr>
      <td><code>token</code></td>
      <td>yes</td>
      <td></td>
    </tr>
    <tr><td colspan="3">
      GitHub token for logging into the ghcr container registry
    </td></tr>
    <tr>
      <td><code>tags</code></td>
      <td>yes</td>
      <td></td>
    </tr>
    <tr><td colspan="3">
      List of image tags (newline separated) for the resulting container image
    </td></tr>
    <tr>
      <td><code>push</code></td>
      <td>no</td>
      <td><code>false</code></td>
    </tr>
    <tr><td colspan="3">
      If true, the image will be pushed to the registry
    </td></tr>
    <tr>
      <td><code>platforms</code></td>
      <td>no</td>
      <td><code>""</code></td>
    </tr>
    <tr><td colspan="3">
      Comma separated list of platforms to build the image for, i.e. <code>linux/amd64,linux/arm64</code>. If left empty, will only build for the native platform.
    </td></tr>
    <tr>
      <td><code>context</code></td>
      <td>no</td>
      <td><code>.</code></td>
    </tr>
    <tr><td colspan="3">
      Context directory for the container image build
    </td></tr>
    <tr>
      <td><code>build-args</code></td>
      <td>no</td>
      <td><code>""</code></td>
    </tr>
    <tr><td colspan="3">
      List of build arguments (newline separated) to be inserted in the container image build
    </td></tr>
    <tr>
      <td><code>file</code></td>
      <td>no</td>
      <td><code>Dockerfile</code></td>
    </tr>
    <tr><td colspan="3">
      Name of the dockerfile to build
    </td></tr>
    <tr>
      <td><code>no-cache</code></td>
      <td>no</td>
      <td><code>true</code></td>
    </tr>
    <tr><td colspan="3">
      Set to false to enable caching of docker layers
    </td></tr>
    <tr>
      <td><code>pull</code></td>
      <td>no</td>
      <td><code>true</code></td>
    </tr>
    <tr><td colspan="3">
      Pull base images from the registry when not available locally
    </td></tr>
  </tbody>
</table>

### Example usage
Below, you will find a working example where we either just build (and not push)
a container image on a pull request, and we fully build and push a container on
the main branch, these two require separate permissions. We'll also add a build
matrix to allow multiple images to be generated:

* `.github/workflows/docker.yml`
  ```yaml
  name: Docker

  on:
    workflow_call:

  jobs:
    docker:
      strategy:
        matrix:
          include:
            - version: trixie
              latest: false
              alt: testing
            - version: bookworm
              latest: true
              alt: stable
            - version: bullseye
              latest: false
              alt: oldstable
      steps:
        - uses: actions/checkout@v4
        - name: Build container image
          uses: tweedegolf/build-container-image@main
          with:
            token: ${{ secrets.GITHUB_TOKEN }}
            push: ${{ github.ref == 'refs/heads/main' }}
            platforms: "linux/amd64,linux/arm64"
            build-args: |
              DEBIAN_VERSION=${{matrix.version}}
            tags: |
              ghcr.io/tweedegolf/debian:${{matrix.version}}
              ghcr.io/tweedegolf/debian:${{matrix.alt}}
              ${{ matrix.latest && 'ghcr.io/tweedegolf/debian:latest' || '' }}
  ```
  This file will be re-used by the two workflows below. As such we only trigger
  it on `workflow_call`. Note how we use a matrix to build multiple images. Of
  course for this workflow to run we'll also need a Dockerfile to be built, but
  that has been omitted in this example.
* `.github/workflows/build-push.yml`
  ```yaml
  name: Build and push

  permissions:
    contents: read
    packages: write

  on:
    push:
      branches:
        - main
    schedule:
      - cron: '30 2 * * SUN'

  jobs:
    build-and-push:
      uses: ./.github/workflows/docker.yml
  ```
  This build and push workflow for the main branch also runs on a schedule every
  week to keep the image up to date. Note how we require package write
  permission in this workflow.

* `.github/workflows/check.yml`
  ```yaml
  name: Checks

  permissions:
    contents: read

  on:
    pull_request:

  jobs:
    build:
      uses: ./.github/workflows/docker.yml
      secrets: inherit
  ```
  The checks workflow will just need read permissions for the repository with
  no write permissions required. We only run it for a pull request.
