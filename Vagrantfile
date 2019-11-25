# Eirini + SCF on microk8s

# PC requirements:
# - CPU: 4-8 cores
# - RAM: 16 GiB
# - Disk: 100 GiB free space

# Notes:
# - Make sure `VBoxHeadless` process is not swapping out. If it does try to lower the requested memory for the VM.

eirini_version = "master"
microk8s_ip = "192.168.51.101"
helm_version = "2.16/stable"
k8s_version = "1.15/stable"
dns_forwarders = ["8.8.8.8", "8.8.4.4"]
enable_rbac = true

# VM configuration
cpus   = 4    # Cores
memory = 8    # GiB
swap   = 8    # GiB
disk   = 120  # GiB

variables = <<~SHELL
  MICROK8S_IP="#{microk8s_ip}"
  EIRINI_VERSION="master"
  K8S_VERSION="#{k8s_version}"
  HELM_VERSION="#{helm_version}"
  EIRINI_DIR="/home/vagrant/eirini"
  DNS_FORWARDERS="#{dns_forwarders.join(" ")}"
  ENABLE_RBAC="#{enable_rbac}"
  SWAP="#{swap}"
SHELL

scripts_common = <<~'SHELL'
  set -euo pipefail
  trap exit_message EXIT

  echo_yellow () {
    local message=$@
    echo -e "\e[1;93m${message}\e[0m"
  }

  exit_message () {
    if [[ $? -ne 0 ]]; then
      echo -e "The provisioning has been interrupted. Please try again:\n\n  vagrant up --provision\n " >&2
    fi
  }

  run_once () {
    local cmd=${1^^}

    date +%FT%T
    if [[ -f "$DONE_DIR/$cmd" ]]; then
      echo "INFO: run_once $cmd - executed before, skipping"
    else
      echo_yellow "INFO: run_once $cmd - executing ..."
      "$@"
      touch "$DONE_DIR/$cmd"
    fi
  }

  retry () {
    local retries=$1; shift
    local command=("$@")
    local output

    for ((counter=1; counter<=retries; counter++)); do
      if output=$("${command[@]}" 2>&1); then
        break
      else
        sleep 1
        echo "Retrying: \"${command[@]}\" ($counter / $retries) ..." >&2
        continue
      fi
      return 1
    done

    echo "$output"
  }

  kubectl () {
    local kubectl_cmd
    kubectl_cmd=$(which kubectl)

    # Make sure the API server is up and running
    retry 5 "$kubectl_cmd" version > /dev/null

    "$kubectl_cmd" "$@"
  }
SHELL

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"
  config.vm.network "private_network", ip: microk8s_ip

  config.vagrant.plugins = ["vagrant-disksize"]
  config.disksize.size = disk.to_s + 'GB'

  config.vm.provider "virtualbox" do |v|
    v.cpus = cpus
    v.memory = memory * 1024
    v.customize ["storagectl", :id, "--name", "SCSI", "--hostiocache", "on"]
  end

  config.vm.provision "shell", name: "system", upload_path: "/tmp/vagrant-shell-system", reset: true do |s|
    s.inline = variables + scripts_common + <<~'SHELL'
      DONE_DIR="$EIRINI_DIR/done"

      prepare () {
        # Prepare directory
        mkdir -p "$EIRINI_DIR" "$DONE_DIR"
        chown vagrant:vagrant "$EIRINI_DIR" "$DONE_DIR"

        # Install some utils
        export DEBIAN_FRONTEND=noninteractive
        apt-get update
        apt-get install -y pwgen jq

        # Install cf cli
        wget -q 'https://cli.run.pivotal.io/stable?release=debian64&source=github' -O /tmp/cf-cli-installer_latest_x86-64.deb
        dpkg -i /tmp/cf-cli-installer_latest_x86-64.deb

        # Add swapfile
        local swapfile="/eirini_swapfile"
        fallocate -l "${SWAP}G" "$swapfile"
        chmod 600 "$swapfile"
        mkswap "$swapfile"
        echo "$swapfile none swap sw 0 0" | tee -a /etc/fstab
        swapon -a
      }

      install_microk8s_and_helm () {
        # Install microk8s
        snap install microk8s --classic --channel="$K8S_VERSION"

        # Install helm
        snap install helm --classic --channel=$HELM_VERSION 

        # Alias microk8s.kubectl -> kubectl
        snap alias microk8s.kubectl kubectl

        # Allow kubectl to be run by user vagrant
        usermod -a -G microk8s vagrant

        # Enable privileged containers
        microk8s.stop
        echo '--allow-privileged=true' | tee -a /var/snap/microk8s/current/args/kube-apiserver
      }

      generate_bits_certificate () {
        # Generate self-signed certificate for registry and install on the host vm
        openssl req -x509 -newkey rsa:4096 -keyout "$EIRINI_DIR/bits_tls.key" -out "$EIRINI_DIR/bits_tls.crt" -days 3650 -nodes \
          -subj "/C=XX/ST=XXXX/L=XXXX/O=XXXX/OU=XXXX/CN=registry.$MICROK8S_IP.nip.io/emailAddress=microk8s@example.com" 2>&1 | tr '\n' ' '
        # Have to install the certificage before containerd starts
        cp "$EIRINI_DIR/bits_tls.crt" /usr/local/share/ca-certificates/
        update-ca-certificates
        chown vagrant:vagrant "$EIRINI_DIR/bits_tls.key" "$EIRINI_DIR/bits_tls.crt"
      }

      start_microk8s () {
        local enable_rbac=$1

        # Start microk8s
        microk8s.start

        # Enable dns, storage and metrics-server
        microk8s.enable dns
        # WORK AROUND: wait for the cluster to be functional after enabling the first plugin
        sleep 5 && microk8s.status --wait-ready --timeout 60
        microk8s.enable storage
        microk8s.enable metrics-server

        # Work around for an issue in Helm https://github.com/helm/helm/issues/6361
        retry 5 kubectl wait apiservice v1beta1.metrics.k8s.io --for=condition=Available --timeout=5m
        sleep 10
        retry 5 kubectl wait apiservice v1beta1.metrics.k8s.io --for=condition=Available --timeout=5m

        # Enable rbac
        if [[ $enable_rbac == true ]]; then
          microk8s.enable rbac
        fi
      }

      main () {
        run_once prepare
        run_once install_microk8s_and_helm
        run_once generate_bits_certificate
        run_once start_microk8s "$ENABLE_RBAC"
      }

      main "$@"
    SHELL
  end

  config.vm.provision "shell", name: "user", upload_path: "/tmp/vagrant-shell-user", privileged: false do |s|
    s.inline = variables + scripts_common + <<~'SHELL'
      DONE_DIR="$EIRINI_DIR/done"

      configure_dns_forwarders () {
        # Update coredns settings
        # FIXME: This is a not reliable way of configuring dns and may not work sometimes, see https://github.com/ubuntu/microk8s/issues/543
        local dns_patch
        dns_patch=$(kubectl -n kube-system get configmap/coredns -o jsonpath='{.data.Corefile}' | sed "s/\(forward .\) 8.8.8.8 8.8.4.4/\1 $DNS_FORWARDERS/" | jq -sR '{"data":{"Corefile":.}}')
        kubectl -n kube-system patch configmap/coredns --type merge -p "$dns_patch"

        # Delete coredns pod to make sure the new settings are applied (sometimes coredns gets stuck in a failed state and not restarted)
        kubectl -n kube-system delete pod -l k8s-app=kube-dns
      }

      helm_init () {
        local enable_rbac=$1

        # Initialize helm and wait for tiller to start (this will deploy tiller to k8s)
        if [[ $enable_rbac == true ]]; then
          kubectl create serviceaccount tiller --namespace kube-system
          kubectl create clusterrolebinding tiller --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
          helm init --wait --history-max 200 --service-account tiller
        else
          helm init --wait --history-max 200
        fi
        helm repo remove local >/dev/null || true
      }

      prepare_values_for_eirini () {
        # Get values.yaml and replace placeholders
        curl -s https://raw.githubusercontent.com/cloudfoundry-incubator/eirini-release/master/values.yaml -o values.yaml
        sed -i "s/<worker-node-ip>/$MICROK8S_IP/g
                s/<persistent-storage-class>/microk8s-hostpath/g
                s/<shared-storage-class>/microk8s-hostpath/g
                s/<cluster-admin-pw>/$(pwgen -nc 12)/g
                s/<uaa-admin-secret>/$(pwgen -nc 12)/g
                s/<blobstore-pw>/$(pwgen -nc 12)/g
                s/<bits-secret>/$(pwgen -nc 12)/g
                s/<bits-user-pw>/$(pwgen -nc 12)/g
               " values.yaml
        cat values.yaml
        echo " "

        local unhandled_placeholders
        unhandled_placeholders=$(grep -Eo "<.+>" values.yaml) || true
        if [[ -n $unhandled_placeholders ]]; then
          {
            echo "ERROR: values.yaml contains unhandled placeholders, please update $FUNCNAME function in Vagrantfile:"
            echo "$unhandled_placeholders"
            echo " "
          } >&2
          exit 1
        fi
      }

      deploy_uaa () {
        ## Install UAA
        helm repo add eirini https://cloudfoundry-incubator.github.io/eirini-release
        helm install eirini/uaa --namespace uaa --name uaa --values values.yaml

        # Wait for secrets to be generated
        kubectl -n uaa wait jobs.batch --all --for condition=Complete --timeout=1h
      }

      deploy_eirini () {
        ## Get secrets
        SECRET=$(kubectl get pods --namespace uaa -o jsonpath='{.items[?(.metadata.name=="uaa-0")].spec.containers[?(.name=="uaa")].env[?(.name=="INTERNAL_CA_CERT")].valueFrom.secretKeyRef.name}')
        CA_CERT="$(kubectl get secret "$SECRET" --namespace uaa -o jsonpath="{.data['internal-ca-cert']}" | base64 --decode -)"
        BITS_TLS_CRT="$(cat bits_tls.crt)"
        BITS_TLS_KEY="$(cat bits_tls.key)"

        # Install Eirini
        helm repo add bits https://cloudfoundry-incubator.github.io/bits-service-release/helm
        git clone https://github.com/cloudfoundry-incubator/eirini-release && git -C "eirini-release" checkout "$EIRINI_VERSION"
        helm install --dep-up eirini-release/helm/cf --namespace scf --name scf --values values.yaml --set "secrets.UAA_CA_CERT=${CA_CERT}" --set "bits.secrets.BITS_TLS_KEY=${BITS_TLS_KEY}" --set "bits.secrets.BITS_TLS_CRT=${BITS_TLS_CRT}"

        local admin_pass
        admin_pass=$(kubectl -n scf get secrets secrets -o jsonpath='{.data.cluster-admin-password}' | base64 -d)

        echo
        echo        'Get to know which components have not started yet'
        echo_yellow "  vagrant ssh -c 'watch -n 10 \"kubectl get pods -A | grep -vE \\\"Completed|([0-9]+)/\1\\\"\"'"
        echo
        echo        'Login to Cloud Foundry:'
        echo_yellow "  cf login --skip-ssl-validation -a api.$MICROK8S_IP.nip.io -u admin -p \"$admin_pass\""
        echo
        echo        'Create space and change target:'
        echo_yellow '  cf create-space myspace && cf target -o system -s myspace'
        echo
        echo        'Push a sample application:'
        echo_yellow '  cd $(mktemp -d) && echo "Hi, Eirini!" > index.html && cf push hello -b staticfile_buildpack'
      }

      main () {
        # Switch to Eirini directory
        cd "$EIRINI_DIR"
        run_once configure_dns_forwarders
        run_once helm_init "$ENABLE_RBAC"
        run_once prepare_values_for_eirini
        run_once deploy_uaa
        run_once deploy_eirini
      }

      main "$@"
    SHELL
  end
end
