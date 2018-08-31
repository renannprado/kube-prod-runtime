#!groovy

// Assumed jenkins plugins:
// - ansicolor
// - custom-tools-plugin
// - pipeline-utility-steps (readJSON)
// - kubernetes
// - jobcacher
// - azure-credentials

// Force using our pod
//def label = UUID.randomUUID().toString()
def label = env.BUILD_TAG.replaceAll(/[^a-zA-Z0-9-]/, '-').toLowerCase()

def withGo(Closure body) {
    container('go') {
        withEnv([
            "GOPATH+WS=${env.WORKSPACE}",
            "PATH+GOBIN=${env.WORKSPACE}/bin",
            "HOME=${env.WORKSPACE}",
        ]) {
            body()
        }
    }
}

def runIntegrationTest(String platform, String kubeprodArgs, String ginkgoArgs, Closure setup) {
    timeout(120) {
        // Regex of tests that are temporarily skipped.  Empty-string
        // to run everything.  Include pointers to tracking issues.
        def skip = ''

        withEnv(["KUBECONFIG=${env.WORKSPACE}/.kubeconf"]) {

            setup()

            withEnv(["PATH+KTOOL=${tool 'kubectl'}"]) {
                sh "kubectl version; kubectl cluster-info"

                unstash 'release'
                unstash 'manifests'

                sh "kubectl --namespace kube-system get po,deploy,svc,ing"

                // install
                // FIXME: we should have a better "test mode", that uses
                // letsencrypt-staging, fewer replicas, etc.  My plan is
                // to do that via some sort of custom jsonnet overlay,
                // since power users will want similar flexibility.

                sh "./release/kubeprod -v=1 install aks --platform=${platform} --manifests=manifests ${kubeprodArgs}"

                // Wait for deployments to rollout before we start the integration tests
                try {
                    timeout(time: 30, unit: 'MINUTES') {
                        sh '''
set +x
for deploy in $(kubectl --namespace kube-system get deploy --output name)
do
  echo "Waiting for rollout of ${deploy}..."
  while ! $(kubectl --namespace kube-system rollout status ${deploy} --watch=false | grep -q "successfully rolled out")
  do
    sleep 3
  done
done
'''
                    }
                } catch (error) {
                    sh "kubectl --namespace kube-system get po,deploy,svc,ing"
                    throw error
                }

                sh 'go get github.com/onsi/ginkgo/ginkgo'
                unstash 'tests'
                dir('tests') {
                    try {
                        ansiColor('xterm') {
                            sh "ginkgo -v --tags integration -r --randomizeAllSpecs --randomizeSuites --failOnPending --trace --progress --slowSpecThreshold=300 --compilers=2 --nodes=4 --skip '${skip}' -- --junit junit --description '${platform}' --kubeconfig ${KUBECONFIG} ${ginkgoArgs}"
                        }
                    } catch (error) {
                        sh "kubectl --namespace kube-system get po,deploy,svc,ing"
                        input 'Paused for manual debugging'
                            throw error
                    } finally {
                        junit 'junit/*.xml'
                    }
                }
            }
        }
    }
}


podTemplate(
    cloud: 'kubernetes-cluster',
    label: label,
    idleMinutes: 1,  // Allow some best-effort reuse between successive stages
    yaml: """
apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 1000
  containers:
  - name: go
    image: golang:1.10.1-stretch
    stdin: true
    command: ['cat']
    resources:
      limits:
        cpu: 2000m
        memory: 2Gi
      requests:
        # rely on burst CPU
        cpu: 10m
        # but actually need ram to avoid oom killer
        memory: 1Gi
  - name: az
    image: microsoft/azure-cli:2.0.30
    stdin: true
    command: ['cat']
    resources:
      limits:
        cpu: 100m
        memory: 500Mi
      requests:
        cpu: 1m
        memory: 100Mi
"""
) {

    env.http_proxy = 'http://proxy.webcache:80/'  // Note curl/libcurl needs explicit :80 !
    // Teach jenkins about the 'go' container env vars
    env.PATH = '/go/bin:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'
    env.GOPATH = '/go'

    stage('Checkout') {
        node(label) {
            checkout scm
            stash includes: '**', excludes: 'tests/**', name: 'src'
            stash includes: 'tests/**', name: 'tests'
        }
    }

    parallel(
        installer: {
            stage('Build') {
                node(label) {
                    withGo() {
                        dir('src/github.com/bitnami/kube-prod-runtime') {
                            timeout(time: 30) {
                                unstash 'src'

                                dir('kubeprod') {
                                    sh 'go version'
                                    sh 'make all'
                                    sh 'make test'
                                    sh 'make vet'

                                    sh 'make release VERSION=$BUILD_TAG'
                                    sh './release/kubeprod --help'
                                    stash includes: 'release/**', name: 'release'
                                }
                            }
                        }
                    }
                }
            }
        },

        manifests: {
            stage('Manifests') {
                node(label) {
                    withGo() {
                        dir('src/github.com/bitnami/kube-prod-runtime') {
                            timeout(time: 30) {
                                unstash 'src'

                                // TODO: use tool, once the next release is made
                                sh 'go get github.com/ksonnet/kubecfg'

                                dir('manifests') {
                                    sh 'make validate KUBECFG="kubecfg -v"'

                                    // Set the default LetsEncrypt issuer to staging in tests to avoid LE ratelimits
                                    // A better way to do this would be to override the default jsonnet config
                                    sh 'sed -i "s/\"default-issuer-name\": .*,/\"default-issuer-name\": $.letsencryptStaging.metadata.name,/" components/cert-manager.jsonnet'
                                }
                                stash includes: 'manifests/**', excludes: 'manifests/Makefile', name: 'manifests'
                            }
                        }
                    }
                }
            }
        })

    def platforms = [:]

    def minikubeKversions = []  // fixme: disabled minikube for now ["v1.8.0", "v1.9.6"]
    for (x in minikubeKversions) {
        def kversion = x  // local bind required because closures
        def platform = "minikube-0.25+k8s-" + kversion[1..3]
        platforms[platform] = {
            stage(platform) {
                node(label) {
                    withGo() {
                        dir('src/github.com/bitnami/kube-prod-runtime') {
                            runIntegrationTest(platform, "", "") {
                                withEnv(["PATH+TOOL=${tool 'minikube'}:${tool 'kubectl'}"]) {
                                    cache(maxCacheSize: 1000, caches: [
                                        [$class: 'ArbitraryFileCache', path: "${env.HOME}/.minikube/cache"],
                                    ]) {
                                        sh 'sudo apt-get -qy update && sudo apt-get install -qy libvirt-clients libvirt-daemon-system virtualbox'
                                        sh "minikube start --kubernetes-version=${kversion}"
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }

    // See:
    //  az aks get-versions -l centralus
    //    --query 'sort(orchestrators[?orchestratorType==`Kubernetes`].orchestratorVersion)'
    def aksKversions = ["1.8.7", "1.9.6"]
    for (x in aksKversions) {
        def kversion = x  // local bind required because closures
        def platform = "aks+k8s-" + kversion[0..2]
        platforms[platform] = {
            stage(platform) {
                node(label) {
                    withGo() {
                        dir('src/github.com/bitnami/kube-prod-runtime') {
                            // NB: `kubeprod` also uses az cli credentials and
                            // $AZURE_SUBSCRIPTION_ID, $AZURE_TENANT_ID.
                            withCredentials([azureServicePrincipal('jenkins-bkpr-owner-sp')]) {
                                def resourceGroup = 'jenkins-bkpr-rg'
                                def clusterName = ("${env.BRANCH_NAME}".take(8) + "-${env.BUILD_NUMBER}-${platform}-" + UUID.randomUUID().toString().take(5)).replaceAll(/[^a-zA-Z0-9-]/, '-').toLowerCase()
                                def dnsPrefix = "${clusterName}"
                                def parentZone = 'tests.bkpr.run'
                                def dnsZone = "${dnsPrefix}.${parentZone}"
                                def adminEmail = "${clusterName}@${parentZone}"
                                def location = "eastus"

                                def aks
                                try {
                                    runIntegrationTest(platform, "--dns-resource-group=${resourceGroup} --dns-zone=${dnsZone} --email=${adminEmail}", "--dns-suffix ${dnsZone}") {
                                        container('az') {
                                            sh '''
az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET -t $AZURE_TENANT_ID
az account set -s $AZURE_SUBSCRIPTION_ID
'''

                                            // Usually, `az aks create` creates a new service
                                            // principal, which is not removed by `az aks
                                            // delete`. We reuse an existing principal here to
                                            // a) avoid this leak b) avoid having to give the
                                            // "outer" principal (above) the power to create
                                            // new service principals.
                                            withCredentials([azureServicePrincipal('jenkins-bkpr-contributor-sp')]) {
                                                def output = sh(returnStdout: true, script: """
az aks create                      \
 --verbose                         \
 --resource-group ${resourceGroup} \
 --name ${clusterName}             \
 --node-count 3                    \
 --node-vm-size Standard_DS2_v2    \
 --location ${location}            \
 --kubernetes-version ${kversion}  \
 --generate-ssh-keys               \
 --service-principal \$AZURE_CLIENT_ID \
 --client-secret \$AZURE_CLIENT_SECRET \
 --tags 'platform=${platform}' 'branch=${BRANCH_NAME}' 'build=${BUILD_URL}'
""")
                                                aks = readJSON(text: output)
                                            }

                                            sh "az aks get-credentials --name ${aks.name} --resource-group ${aks.resourceGroup} --admin --file \$KUBECONFIG"

                                            // create dns zone
                                            sh "az network dns zone create --name ${dnsZone} --resource-group ${resourceGroup} --tags 'platform=${platform}' 'branch=${BRANCH_NAME}' 'build=${BUILD_URL}'"

                                            // update SOA record for quicker updates
                                            sh "az network dns record-set soa update --resource-group ${resourceGroup} --zone-name ${dnsZone} --expire-time 60 --retry-time 60 --refresh-time 60 --minimum-ttl 60"

                                            // update glue records in parent zone
                                            def output = sh(returnStdout: true, script: "az network dns zone show --name ${dnsZone} --resource-group ${resourceGroup} --query nameServers")
                                            for (ns in readJSON(text: output)) {
                                                sh "az network dns record-set ns add-record --resource-group ${resourceGroup} --zone-name ${parentZone} --record-set-name ${dnsPrefix} --nsdname ${ns}"
                                            }

                                            // update TTL for NS record to 60 seconds
                                            sh "az network dns record-set ns update --resource-group ${resourceGroup} --zone-name ${parentZone} --name ${dnsPrefix} --set ttl=60"
                                        }

                                        // Reuse this service principal for externalDNS and oauth2.  A real (paranoid) production setup would use separate minimal service principals here.
                                        withCredentials([azureServicePrincipal('jenkins-bkpr-contributor-sp')]) {
                                            // NB: writeJSON doesn't work without approvals(?)
                                            // See https://issues.jenkins-ci.org/browse/JENKINS-44587

                                            // TODO: The path to kubeprod.json should be passed to `kubeprod` in some way.
                                            // the default behavior is reading it from the cwd.
                                            writeFile([file: 'kubeprod.json', text: """
{
  "dnsZone": "${dnsZone}",
  "contactEmail": "${adminEmail}",
  "externalDns": {
    "tenantId": "${AZURE_TENANT_ID}",
    "subscriptionId": "${AZURE_SUBSCRIPTION_ID}",
    "aadClientId": "${AZURE_CLIENT_ID}",
    "aadClientSecret": "${AZURE_CLIENT_SECRET}",
    "resourceGroup": "${resourceGroup}"
  },
  "oauthProxy": {
    "client_id": "${AZURE_CLIENT_ID}",
    "client_secret": "${AZURE_CLIENT_SECRET}",
    "azure_tenant": "${AZURE_TENANT_ID}"
  }
}
"""
                                            ])
                                        }
                                    }
                                }
                                finally {
                                    if (aks) {
                                        container('az') {
                                            sh '''
az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET -t $AZURE_TENANT_ID
az account set -s $AZURE_SUBSCRIPTION_ID
'''
                                            sh "az network dns record-set ns delete --yes --resource-group ${resourceGroup} --zone-name ${parentZone} --name ${dnsPrefix} || :"
                                            sh "az network dns zone delete --yes --name ${dnsZone} --resource-group ${resourceGroup} || :"
                                            sh "az aks delete --yes --name ${aks.name} --resource-group ${aks.resourceGroup} --no-wait"
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }

    parallel platforms
}