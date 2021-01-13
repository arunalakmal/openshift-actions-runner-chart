# OpenShift Hosted GitHub Action Runners

This repository contains tools for building and deploying containers in an OpenShift cluster that act as [self-hosted GitHub Action runners](https://docs.github.com/en/free-pro-team@latest/actions/hosting-your-own-runners/about-self-hosted-runners).

See [`runners/base`](./runners/base) for the base runner.

See [`runners/buildah`](./runners/buildah) for a Dockerfile which builds on the base runner to add buildah and podman - [with caveats](./runners/buildah/README.md).

The idea is that the base runner [can be extended](#creating-your-own-runner-image) to build larger, more complex images that have additional capabilities.

The images are hosted at [quay.io/redhat-github-actions](https://quay.io/redhat-github-actions/).

## Installing runners using Helm

You can install the runner into your cluster using the Helm chart in this repository.

1. Runners can be scoped to an organization or a repository. Decide what the scope of your runner will be.
2. [Create a GitHub Personal Access Token](https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token) (PAT) which has the `repo` permission scope.
    - The user who created the token must have administrator permission on the repository/organization the runner will be added to.
    - If the runner will be organization-scoped, the token must also have the `admin:org` scope.
    - This will be stored in [a Kubernetes secret](./helm-charts/pat-secret).
    - Make sure the cluster or namespace you are installing into is sufficiently secure, since anyone who can describe the secret or shell into the container can read your token.
3. Clone this repository and `cd` into it:
```bash
git clone git@github.com:redhat-actions/openshift-hosted-runner.git && \
    cd openshift-hosted-runner
```
4. Create the PAT secret.
```bash
helm install runner-pat ./helm-charts/runner-pat-secret \
    --set-string githubPat=$GITHUB_PAT
```
5. Install the deployment helm chart. Leave out `githubRepository` if you want an org-scoped runner.

```bash
helm install runner ./helm-charts/deployment \
    --set-string githubOwner=<GitHub user or org that owns the repo> \
    --set-string githubRepository=<repo to add runner to>
```

6. You can re-run step 5 if you want to scale up the number of runners, optionally with different images, labels, etc.

For other configuration options, see [`values.yaml`](./runner-chart/values.yaml). The [`Dockerfile`](./runners/base/Dockerfile) also has some environment variables you can override.

After the helm install, run `oc get po -w` to make sure your runner pod(s) come up successfully. The runners should show up under `Settings > Actions > Self-hosted runners` shortly afterward.

When the runner pods are terminated for any reason, they should remove themselves from the repository's self-hosted runner list. See [entrypoint.sh](./runners/base/entrypoint.sh).

## Creating your own runner image

You can create your own runner image based on this one, and install any runtimes and tools your workflows need.

1. Create your own Dockerfile, with `FROM quay.io/redhat-github-actions/redhat-actions-runner:<tag>`.
2. Edit the Dockerfile to install and set up your tools, environment, etc. Do not override the `ENTRYPOINT`.
3. Build and push your new runner image.
4. Run the `helm install` as above, but set `image` to your image, and `tag` to your tag.

## Credits
This repository builds on the work done in [bbrowning/github-runner](https://github.com/bbrowning/github-runner), which is forked from [SanderKnape/github-runner](https://github.com/SanderKnape/github-runner).