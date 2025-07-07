# All NFS pods are failed (Error or Unknown)
Example logs:
    leaderelection.go:320] error retrieving resource lock application/k8s-sigs.io-nfs-subdir-external-provisioner: Get "https://10.96.0.1:443/api/v1/namespaces/application/endpoints/k8s-sigs.io-nfs-subdir-external-provisioner": context deadline exceeded
    failed to renew lease monitoring/k8s-sigs.io-nfs-subdir-external-provisioner: timed out waiting for the condition
    leaderelection lost
    
to do: 
ip route show default shows only 1 default route:
`default via 172.27.0.1 dev enp0s8 proto dhcp src 172.27.0.5 metric 20101 `
but we need another one as well. so we need to add it for all worker nodes.

Show active interfaces information:
`nmcli device show`
No default gateway is assigned by VirtualBox's DHCP server for host-only adapters 
At Host machine:
`ip addr show vboxnet0`
sudo ip route add default via 172.28.0.5 dev enp0s3

It can ping nfs server, but still cannot successfully complete: `showmount -e <nfs-server-ip>`
