import java.text.SimpleDateFormat

currentBuild.displayName = new SimpleDateFormat("yy.MM.dd").format(new Date()) + "-" + env.BUILD_NUMBER
env.PROJECT = "go-demo-3"
env.REPO = "https://github.com/innokgoldberg/go-demo-3.git"
env.IMAGE = "innokgoldberg/go-demo-3"
env.IP="192.168.59.139"
env.DOMAIN = "${env.IP}.nip.io"
env.ADDRESS = "go-demo-3.${env.IP}.nip.io"
env.CM_ADDR = "cm.${env.IP}.nip.io"
env.CHART_VER = "0.0.1"
def label = "jenkins-slave-${UUID.randomUUID().toString()}"

def dockerhub_secrets = [
  [path: 'secret/jenkins/dockerhub', engineVersion: 2, secretValues: [
    [envVar: 'DOCKERHUB_USER', vaultKey: 'user'],
    [envVar: 'DOCKERHUB_PASSWORD', vaultKey: 'password']]],
  ]
def chartmuseum_secrets = [
  [path: 'secret/jenkins/chartmuseum', engineVersion: 2, secretValues: [
    [envVar: 'CHARTMUSEUM_USER', vaultKey: 'user'],
    [envVar: 'CHARTMUSEUM_PASSWORD', vaultKey: 'password']]],
]
def configuration = [vaultUrl: 'http://vault.default.svc.cluster.local:8200',  vaultCredentialId: 'vault-approle', engineVersion: 2]

podTemplate(
  label: label,
  namespace: "go-demo-3-build",
  serviceAccount: "build",
  yaml: """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: helm
    image: alpine/helm
    command: ["cat"]
    tty: true
  - name: kubectl
    image: vfarcic/kubectl
    command: ["cat"]
    tty: true
  - name: golang
    image: golang:1.12
    command: ["cat"]
    tty: true
"""
) {
  node(label) {
    node("docker") {
      stage("build") {
        checkout scm
        k8sBuildImageBeta(env.IMAGE,configuration,dockerhub_secrets)
      }
    }
    stage("func-test") {
      try {
        container("helm") {
          checkout scm
          k8sUpgradeBeta(env.PROJECT, env.DOMAIN,"--set replicaCount=2 --set dbReplicaCount=1")
        }
        container("kubectl") {
          k8sRolloutBeta(env.PROJECT)
        }
        container("golang") {
          k8sFuncTestGolang(env.PROJECT, env.DOMAIN)
        }
      } catch(e) {
          error "Failed functional tests"
      } finally {
        container("helm") {
          k8sDeleteBeta(env.PROJECT)
        }
      }
    }
    if ("${BRANCH_NAME}" == "master") {
        stage("release") {
          node("docker") {
            k8sPushImage(env.IMAGE,configuration,dockerhub_secrets)
          }
          container("helm") {
            k8sPushHelm(env.PROJECT, env.CHART_VER, env.CM_ADDR, env.IP,configuration, chartmuseum_secrets)
          }
        }
        stage("deploy") {
          try {
            container("helm") {
              k8sUpgrade(env.PROJECT, env.ADDRESS)
            }
            container("kubectl") {
              k8sRollout(env.PROJECT)
            }
            container("golang") {
              k8sProdTestGolang(env.ADDRESS)
            }
          } catch(e) {
            container("helm") {
              k8sRollback(env.PROJECT)
            }
          }
       }
    }
  }
}