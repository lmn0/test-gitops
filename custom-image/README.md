# Custom repo-server image (alternative Helm 3.7 delivery method)

This directory is **not wired into the GitOps tree by default**. The active mechanism for
requirement 3 (pin Helm to 3.7) is the initContainer + subPath volume mount defined in
`apps/argocd/values.yaml`, which overwrites `/usr/local/bin/helm` inside the stock
`repo-server` container at pod startup.

Use this Dockerfile instead if you need the binary baked in at build time rather than
injected at runtime (air-gapped clusters, stricter supply-chain rules, or if you simply
don't want an internet-fetching initContainer in the pod spec). To switch:

1. Build and push the image (see comments in `Dockerfile`).
2. In `apps/argocd/values.yaml`, delete the `repoServer.initContainers` /
   `repoServer.volumes` / `repoServer.volumeMounts` block.
3. Add:
   ```yaml
   repoServer:
     image:
       repository: your-registry/argocd-repo-server
       tag: v2.13.2-helm3.7.2
   ```
4. Commit and push -- Argo CD will pick up the change against itself (self-management) and
   roll the repo-server deployment.
