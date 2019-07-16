# eirini-on-microk8s
Vagrantfile for Eirini on microk8s is an easy way to get [microk8s](https://microk8s.io) with [SCF](https://github.com/SUSE/scf) and [Eirini](https://github.com/cloudfoundry-incubator/eirini-release) running on your PC.

# How to use
## Setting up
```
git clone https://github.com/giner/eirini-on-microk8s
cd eirini-on-microk8s
# This may take a while. Get your favorite drink and relax.
vagrant up
```
## Accessing CF API
When all components are up and running CF API will be available the following endpoint:
- `https://api.192.168.51.101.nip.io`
- [cf cli](https://github.com/cloudfoundry/cli) can be used either on your PC or from inside the VM. To get inside the VM run `vagrant ssh`.

## Get to know status of the components
```
vagrant ssh -c 'watch -n 10 "kubectl get pods --all-namespaces"'
```

## Notes
- The Eirini is configured to stage on Kubernetes. This Vagrantfile is not tested with staging on Diego and may require more resources.
