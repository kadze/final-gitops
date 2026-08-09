# kadze-gitops

ArgoCD's source of truth for `eks-kadze-dev`. Everything ArgoCD manages on the cluster is
described here — changes land by merging to this repo and letting ArgoCD reconcile, not by
`kubectl apply`.

No CI runs in this repo. Merging is the only deploy action — that's the point.

See `kadze-final-proj-deliverables/extra-stories/self-learn-argocd/` for the learning log
and the reasoning behind how this repo is structured.

## After a cluster recreate

The cluster (`eks-kadze-dev`) gets torn down and rebuilt daily. ArgoCD's own install is
automated (`kadze-platform-tools-25c-ubuntu`'s `deploy-platform-tools.yaml`, which already
runs on every push), so it comes back on its own. Two things it can't do for you, because
they aren't files this repo — or that workflow — tracks:

1. **Re-add the repo credential.** The Secret ArgoCD uses to authenticate to this repo lived
   in the destroyed cluster, so it's gone. The deploy key itself still exists — locally at
   `~/.ssh/kadze_gitops_argocd`, and its public half is still registered as a read-only
   Deploy Key on this repo (that registration lives on GitHub, not in the cluster, so it
   survives). Re-point ArgoCD at it:
   ```bash
   aws eks update-kubeconfig --name eks-kadze-dev --region us-east-1
   kubectl -n argocd port-forward svc/argocd-server 8080:443 &
   argocd login localhost:8080 --username admin --password <current admin password> --insecure
   argocd repo add git@github.com:312school/kadze-gitops.git \
     --ssh-private-key-path ~/.ssh/kadze_gitops_argocd
   ```

2. **Re-apply the `Application`(s).** `Application` CRs live inside the cluster; only their
   *source* — this repo — survives a teardown. Until Stage 5 (app-of-apps) exists, each one
   has to be re-applied once by hand:
   ```bash
   kubectl apply -f applications/hello.yaml
   ```
   Once app-of-apps lands, this collapses to applying a single root `Application`, which then
   pulls in everything else from git on its own.

Deliberately not automated yet: injecting the deploy key from a CI secret, and
auto-applying the root `Application`. Both are real decisions (see the decision log in
`kadze-final-proj-deliverables`), not something to bolt on at the end of a session.
