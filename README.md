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
runs on every push), so it comes back on its own. Three things it can't do for you:

1. **Re-add both repo credentials.** The Secrets ArgoCD uses to authenticate to `kadze-gitops`
   and `kadze-versus` lived in the destroyed cluster, so they're gone. Both deploy keys still
   exist locally (`~/.ssh/kadze_gitops_argocd`, `~/.ssh/kadze_versus_argocd`), and their public
   halves are still registered as read-only Deploy Keys on GitHub (that registration survives —
   it's not in the cluster). Fastest path is recreating the Secrets directly rather than going
   through `argocd repo add` + a port-forwarded login:
   ```bash
   aws eks update-kubeconfig --name eks-kadze-dev --region us-east-1

   kubectl create secret generic repo-kadze-gitops -n argocd \
     --from-literal=type=git \
     --from-literal=url=git@github.com:312school/kadze-gitops.git \
     --from-file=sshPrivateKey=$HOME/.ssh/kadze_gitops_argocd \
     --dry-run=client -o yaml | kubectl label -f - --local -o yaml argocd.argoproj.io/secret-type=repository | kubectl apply -f -

   kubectl create secret generic repo-kadze-versus -n argocd \
     --from-literal=type=git \
     --from-literal=url=git@github.com:312school/kadze-versus.git \
     --from-file=sshPrivateKey=$HOME/.ssh/kadze_versus_argocd \
     --dry-run=client -o yaml | kubectl label -f - --local -o yaml argocd.argoproj.io/secret-type=repository | kubectl apply -f -
   ```

2. **Re-apply `root` and the `learning` AppProject.** These are the only two things nothing
   watches — everything else (`hello`, `podinfo`, `versus-backend`, `versus-frontend`, `foo`/`bar`
   when they exist) is picked up automatically by `root` once it's live:
   ```bash
   kubectl apply -f projects/learning.yaml
   kubectl apply -f root.yaml
   ```

3. **Re-sync `versus/db-credentials` with RDS's actual password.** RDS itself gets destroyed
   and recreated along with the cluster (`rds-mysql` module, `manage_master_user_password =
   true`), which means a **new random master password every time** — the app secret and RDS's
   own auto-managed secret will be out of sync on every fresh cluster until this runs:
   ```bash
   aws rds describe-db-instances --region us-east-1 \
     --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text
   # then, using that ARN:
   aws secretsmanager get-secret-value --secret-id <that ARN> --region us-east-1 \
     --query SecretString --output text | python3 -c "import json,sys; print(json.load(sys.stdin)['password'])" > /tmp/rds_pw.txt
   aws secretsmanager get-secret-value --secret-id versus/db-credentials --region us-east-1 \
     --query SecretString --output text > /tmp/current-secret.json
   python3 -c "
   import json
   pw = open('/tmp/rds_pw.txt').read().strip()
   d = json.load(open('/tmp/current-secret.json'))
   d['MYSQL_PASSWORD'] = pw
   json.dump(d, open('/tmp/updated-secret.json', 'w'))
   "
   aws secretsmanager put-secret-value --secret-id versus/db-credentials --region us-east-1 \
     --secret-string file:///tmp/updated-secret.json
   rm -f /tmp/rds_pw.txt /tmp/current-secret.json /tmp/updated-secret.json
   # then force versus-backend to remount the corrected secret:
   kubectl delete pod -n versus -l app=versus-backend
   ```
   Real architecture gap, not a workaround to remove: production would point the app directly
   at RDS's own managed secret, or have Terraform populate the app secret from the RDS output,
   instead of maintaining two independent secrets that can drift.

Deliberately not automated yet: injecting the deploy keys from CI secrets, auto-syncing the
`versus/db-credentials` password, and auto-applying `root`/the `AppProject`. All real decisions
(see the decision log in `kadze-final-proj-deliverables`), not something to bolt on at the end
of a session.
