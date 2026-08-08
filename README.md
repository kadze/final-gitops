# kadze-gitops

ArgoCD's source of truth for `eks-kadze-dev`. Everything ArgoCD manages on the cluster is
described here — changes land by merging to this repo and letting ArgoCD reconcile, not by
`kubectl apply`.

No CI runs in this repo. Merging is the only deploy action — that's the point.

See `kadze-final-proj-deliverables/extra-stories/self-learn-argocd/` for the learning log
and the reasoning behind how this repo is structured.
