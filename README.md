# STEPS

After forking this repo, follow the steps below to setup a working demo of FluxCD:

### Install fluxctl

   <https://fluxcd.io/flux/installation/>  

### Bootstrap Flux

   ```bash
    flux install --components-extra=image-reflector-controller,image-automation-controller --export > gotk-components.yaml
   ```

   The generated file contains the manifests for the Flux components and the HelmRelease for the Helm Operator, also the image-automation-controller and image-reflector-controller components.

   Apply the manifests 👇

   ```bash
   kubectl apply -f gotk-components.yaml
   ```

### Create the ssh-credentials secret

   FluxCD needs the ssh-credentials secret to access the git repository. The secret should contain the private key, the public key and the known_hosts file.

- Generate the ssh keys

   ```bash
   ssh-keygen -t rsa -b 4096 -C "abdelattie@gmail.com" 
   ```

- Add the public key to your git repository deploy keys [ settings > deploy keys > add deploy key ]

- Create the secret in the cluster

   ```bash
   kubectl create secret generic ssh-credentials -n flux-system \
   --from-file=identity=./ssh-deploy-key \
   --from-file=identity.pub=./ssh-deploy-key.pub \
   --from-literal=known_hosts="github.com ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCj7ndNxQowgcQnjshcLrqPEiiphnt+VTTvDP6mHBL9j1aNUkY4Ue1gvwnGLVlOhGeYrnZaMgRK6+PKCUXaDbC7qtbW8gIkhL7aGCsOr/C56SJMy/BCZfxd1nWzAOxSDPgVsmerOBYfNqltV9/hWCqBywINIR+5dIg6JTJ72pcEpEjcYgXkE2YEFXV1JHnsKgbLWNlhScqb2UmyRkQyytRLtL+38TGxkxCflmO+5Z8CSSNY7GidjMIZ7Q4zMjA2n1nGrlTDkzwDCsw+wqFPGQA179cnfGWOWRVruj16z6XyvxvjJwbz0wQZ75XK5tKSb7FNyeIEs4TT4jk+S4dhPeAUC5y+bDYirYgM4GC7uEnztnZyaVWQ7B381AK4Qdrwt51ZqExKbQpTUNn+EjqoTwvqNj4kqx5QUCI0ThS/YkOxJCXmPUWZbhjpCg56i+2aB6CmK2JGhn57K5mj0MNdBXA4/WnwH6XoPWJzK5Nyu2zB3nAZp+S5hpQs+p1vN1/wsjk="
   ```

### Setup FluxCD sync: Create FluxCD GitRepository and Kustomization

- check the file `gotk-components.yaml` for the `GitRepository` and `Kustomization` resources
- edit the `url` field in the `GitRepository` resource to point to your forked repository
- edit the path field in the `Kustomization` resource to point to the `kustomization` directory in your forked repository

 Apply the manifests and watch for errors

   ```bash
   kubectl apply -f gotk-components.yaml
   ```

   Now we are ready to start adding our Applications definitions in the folder ynov, so that FluxCD can start reconciling the state of the cluster with the desired state defined in the git repository.

### let's create a deployment or our movies application

- navigate to the backend directory in this repo.
- There is a deployment, configmap and service example in the directory.
- revise them and remember the folder `synced/backend`, we are going to use it in the next steps.

### Flux Image Automation

> Make sure to use the SAMPLES provided in `Samples` folder to create the resources
We need to add few flux components to our cluster to enable image automation.

after that fluxCD will be able to:

- watch for new images tags in the docker repository.
- update the deployment manifest with the new image tag, commit and push the manifest with the new image tag it to the gitops repository.
- apply the new manifests with updated image to the cluster.

let's create image repository and image automation for our movies application

- first we need to have a docker registry, you can use dockerhub or any other registry.
- we need to enable create a secret with the docker registry credentials, fluxCD will use these credentials to pull the images from the registry.
  
```bash
kubectl create secret docker-registry regcred -n flux-system \
   --docker-server=https://index.docker.io/v1/ \
   --docker-username=random9deploy \
   --docker-password=random9deploy \
   --docker-email=thegeekstudent@gmail.com
```

- create the image repository and image automation resources, `ImageRrepo, ImageAutomation, ImagePolicy`.

- let's apply these resources to the cluster.

```bash
kubectl apply -k synced/flux
```

- Inspect the resources inside the cluster

- edit the backend deployment and add the following comment after the image tag, this will tell fluxCD to use the image policy we defined in the image policy resource.
`# {"$imagepolicy": "flux-system:backend-image-policy"}`

- commit and push the changes to the gitops repository.

- add a new tag to the image in the docker registry, and watch fluxCD updating the deployment manifest with the new image tag.
