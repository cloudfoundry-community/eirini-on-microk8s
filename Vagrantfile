# Eirini + SCF on microk8s

# Requirements:
# - CPU: 4-8 cores
# - RAM: 16 GiB
# - Disk: 100 GiB free space

# Notes:
# - Make sure `VBoxHeadless` process is not swapping out. If it does try to lower the requested memory for the VM.

microk8s_ip = "192.168.51.101"
eirini_version = "master"
k8s_version = "1.15/stable"
eirini_dir = "/home/vagrant/eirini"
dns_forwarders = ["8.8.8.8", "8.8.4.4"]

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
SHELL

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"
  config.vm.network "private_network", ip: microk8s_ip

  config.vagrant.plugins = ["vagrant-disksize"]
  config.disksize.size = '120GB'

  config.vm.provider "virtualbox" do |v|
    v.cpus = 4
    v.memory = 8192
    v.customize ["storagectl", :id, "--name", "SCSI", "--hostiocache", "on"]
  end

  config.vm.provision "shell", name: "system" do |s|
    s.args = [microk8s_ip, k8s_version, eirini_dir]

    s.inline = scripts_common + <<~'SHELL'
      MICROK8S_IP=$1
      K8S_VERSION=$2
      EIRINI_DIR=$3

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
        fallocate -l 8G "$swapfile"
        chmod 600 "$swapfile"
        mkswap "$swapfile"
        echo "$swapfile none swap sw 0 0" | tee -a /etc/fstab
        swapon -a
      }

      install_microk8s_and_helm () {
        # Install microk8s and helm
        snap install microk8s --classic --channel="$K8S_VERSION"
        snap install helm --classic

        # Alias microk8s.kubectl -> kubectl
        snap alias microk8s.kubectl kubectl

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
        # Start microk8s
        microk8s.start

        # Enable dns, storage and metrics-server
        microk8s.enable storage
        microk8s.enable dns
        #microk8s.enable metrics-server
      }

      main () {
        run_once prepare
        run_once install_microk8s_and_helm
        run_once generate_bits_certificate
        run_once start_microk8s
      }

      main "$@"
    SHELL
  end

  config.vm.provision "shell", name: "user", privileged: false do |s|
    s.args = [microk8s_ip, eirini_version, eirini_dir, dns_forwarders.join(" ")]

    s.inline = scripts_common + <<~'SHELL'
      MICROK8S_IP=$1
      EIRINI_VERSION=$2
      EIRINI_DIR=$3
      DNS_FORWARDERS=$4

      DONE_DIR="$EIRINI_DIR/done"

      kubectl () {
        local kubectl_cmd counter
        kubectl_cmd=$(which kubectl)

        # Make sure the API server is up and running
        local retries=5
        for ((counter=1; counter<=retries; counter++)); do
          if "$kubectl_cmd" version > /dev/null; then
            break
          else
            sleep 1
            echo "Retrying connection to the API server ($counter / $retries) ..." >&2
            continue
          fi
          return 1
        done

        "$kubectl_cmd" "$@"
      }

      configure_dns_forwarders () {
        # Update coredns settings
        local dns_patch
        dns_patch=$(kubectl -n kube-system get configmap/coredns -o jsonpath='{.data.Corefile}' | sed "s/\(forward .\) 8.8.8.8 8.8.4.4/\1 $DNS_FORWARDERS/" | jq -sR '{"data":{"Corefile":.}}')
        kubectl -n kube-system patch configmap/coredns --type merge -p "$dns_patch"
      }

      deploy_heapster () {
        # Install heapster
        git clone https://github.com/kubernetes-retired/heapster
        pushd heapster/deploy
          ./kube.sh start
        popd
      }

      helm_init () {
        # Initialize helm and wait for tiller to start (this will deploy tiller to k8s)
        helm init --wait --history-max 200
        helm repo remove local >/dev/null || true
      }

      prepare_values_for_eirini () {
        # Install Eirini
        ## Get values.yaml and replace placeholders
        ## FIXME: Templating here pleeeaaassseee
        curl -s https://raw.githubusercontent.com/cloudfoundry-incubator/eirini-release/master/values.yaml -o values.yaml
        sed -i "s/<worker-node-ip>/$MICROK8S_IP/g
                s/<your-storage-class>/microk8s-hostpath/g
                s/\(CLUSTER_ADMIN_PASSWORD:\) REPLACE/\1 $(pwgen -nc 12)/
                s/\(UAA_ADMIN_CLIENT_SECRET:\) REPLACE/\1 $(pwgen -nc 12)/
                s/\(BLOBSTORE_PASSWORD: &BLOBSTORE_PASSWORD\) \"REPLACE\"/\1 $(pwgen -nc 12)/
                s/\(BITS_SERVICE_SECRET:\) REPLACE/\1 $(pwgen -nc 12)/
                s/\(BITS_SERVICE_SIGNING_USER_PASSWORD:\) REPLACE/\1 $(pwgen -nc 12)/
                s/\(ENABLE_OPI_STAGING:\) false/\1 true/
               " values.yaml
        cat values.yaml
        echo
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

        helm repo add bits https://cloudfoundry-incubator.github.io/bits-service-release/helm

        git clone -b "$EIRINI_VERSION" https://github.com/cloudfoundry-incubator/eirini-release
        helm install --dep-up eirini-release/helm/cf --namespace scf --name scf --values values.yaml --set "secrets.UAA_CA_CERT=${CA_CERT}" --set "eirini.secrets.BITS_TLS_KEY=${BITS_TLS_KEY}" --set "eirini.secrets.BITS_TLS_CRT=${BITS_TLS_CRT}" --set "sizing.locket.capabilities={ALL}"

        local admin_pass
        admin_pass=$(kubectl -n scf get secrets secrets -o jsonpath='{.data.cluster-admin-password}' | base64 -d)

        echo
        echo        'Login to Cloud Foundry:'
        echo_yellow "  cf login --skip-ssl-validation -a api.$MICROK8S_IP.nip.io -u admin -p \"$admin_pass\""
        echo
        echo        'Create space and change target:'
        echo_yellow '  cf create-space myspace && cf target -o system -s myspace'
        echo
        echo        'Push a sample application:'
        echo_yellow '  cd $(mktemp -d); echo "Hi, Eirini!" > index.html; cf push hello -b staticfile_buildpack'
      }

      main () {
        # Switch to Eirini directory
        cd "$EIRINI_DIR"
        run_once configure_dns_forwarders
        run_once deploy_heapster
        run_once helm_init
        run_once prepare_values_for_eirini
        run_once deploy_uaa
        run_once deploy_eirini
      }

      main "$@"
    SHELL
  end
end
