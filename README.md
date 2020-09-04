# eirini-on-microk8s
Vagrantfile for Eirini on microk8s is an easy way to get Cloud Foundry running on top of Kubernetes in a single VM running on your PC ([microk8s](https://microk8s.io) + [SCF](https://github.com/SUSE/scf) + [Eirini](https://github.com/cloudfoundry-incubator/eirini-release))

# How to use
## Setting up
```
git clone https://github.com/cloudfoundry-community/eirini-on-microk8s
cd eirini-on-microk8s
# Make sure you have a recent version of Vagrant which supports automatic installation for plugins.
vagrant up
```

## Get to know which components have not started yet
```
# The setup may take a while. Get your favorite drink and relax.
vagrant ssh -c 'watch -n 10 "kubectl get pods -A | grep -vE \"Completed|([0-9]+)/\1\""'
```

## Accessing CF API
When all components are up and running CF API will be available the following endpoint:
- `https://api.192.168.51.101.nip.io`
- [cf cli](https://github.com/cloudfoundry/cli) can be used either on your PC or from inside the VM. To get inside the VM run `vagrant ssh`.

## Cleaning up
```
# When you are done or messed up
vagrant destroy
```

## Notes
- By default the latest version (master branch) of [eirini-release](https://github.com/cloudfoundry-incubator/eirini-release) is deployed which, sometimes, may be not fully functional.
  Version of Eirini release can be changed in Vagrantfile, e.g. `eirini_version = "1.8.0"`
- This setup has Eirini configured to stage on Kubernetes (which is default). This is not being tested with staging on Diego therefore may not work or require more resources.
