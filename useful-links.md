Workshop that helped me configure TRAEFIK ingress:
https://github.com/traefik-workshops/traefik-workshop/tree/master/exercise-7

Kubernetes install via manifests JENKINS:
https://www.jenkins.io/doc/book/installing/kubernetes/#create-a-volume

Also, an ephemeral Kubernetes pod-based Jenkins agent is a great way to reduce the cost of the CI environment, as Jenkins agents are spun up only if there is a build request.

NFS provisioner setup
https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/master/deploy

Vault initial set up:
https://akjamie.github.io/post/2023-03-25-vault-on-k8s/

Netshoot with Kubernetes

if you want to debug using an ephemeral container in an existing pod:

$ kubectl debug mypod -it --image=nicolaka/netshoot

if you want to spin up a throw away pod for debugging.

$ kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot
