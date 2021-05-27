pipeline {
  agent any

  parameters {
    string(name: repo, description: "repository url.")
  }

  stages {
    stage('Download kubeval') {
      when { not fileExists file:'kubeval' }
      steps {
        sh """wget -O kubeval.tar.gz --quiet ${KUBEVAL_URL}"""
        sh """tar xf kubeval.tar.gz"""
        sh """chmod 755 ./kubeval"""
      }
    }
    stage('Download kubeval') {
      when { not fileExists file:'helm' }
      steps {
        sh """wget -O helm.tar.gz --quiet ${KUBEVAL_URL}"""
        sh """tar xf helm.tar.gz"""
        sh """cp -vf ./linux-amd64/helm ."""
      }
    }
    stage('Download kubeval') {
      when { not fileExists file:'kustomize' }
      steps {
        sh """wget -O kustomize.tar.gz --quiet ${KUSTOMIZE_URL}"""
        sh """tar xf kustomize.tar.gz"""
        sh """chmod 755 kustomize"""
      }
    }
    stage('Checkout') {
      steps {
        dir('src') {

        }
      }
    }
    stage('Build Templates') {
      steps {
        script {
          // Generate templates
          def ko = new KubeObjects('src/manifest.yml')
          ko.parse()
          def hcb = new HelmCommandBuilder()
          hcb.buildDeploymentCommand(ko)
          hcb.buildIngressCommand(ko)
          dir($WORKSPACE) {
            // find manifest
            def files = findFiles(glob: 'src/**.yml')
            if (files.size() > 0) {
              sh """${hcb.getCommand()} --set manifest=${files[0].path} > kustomization.yml"""
              sh """kustomize build > ${files[0].path}"""
              sh """kubeval ${files[0].path} --strict --schema-location https://raw.githubusercontent.com/yannh/kubernetes-json-schema/master"""
            } else {
              error """Cannot find any manifests under 'src'."""
            }          
          }
        }
      }
    }
    stage('Commits') {
      steps {
        sh """git add ."""
        sh """git commit -m "Batch updates." """
        sh """git push -u --set-upstream origin ${params.branch}"""
      }
    }
    stage('Create PR') {
      steps {
        sh """createGitHubPR ..."""
      }
    }
  }
}


class Ingress {
    String name
    String serviceName
    String servicePort
}

class Deployment {
    String name
}

class KubeObjects {
    final private String manifest
    def ingresses = []
    def deployments = []

    KubeObjects(String fire) {
        this.manifest = fire
        this.ingresses = []
        this.deployments = []
    }

    def getIngresses() {
        return this.ingresses
    }

    def getDeployments() {
        return this.deployments
    }

    def parse() {
        def data = readYaml file:this.manifest
        for(Object obj: data) {
            if (obj.kind == 'Ingress') {
                String name = obj.metadata.name
                String backendSvcName = obj.spec.rules[0].http.paths[0].backend.serviceName
                if (backendSvcName == null) {
                    throw new Exception("Cannot find 'serviceName' at path /spec/rules/0/http/paths/0/backend/serviceName")
                }
                String backendSvcPort = obj.spec.rules[0].http.paths[0].backend.servicePort
                if (backendSvcPort == null) {
                    throw new Exception("Cannot find 'servicePort' at path /spec/rules/0/http/paths/0/backend/servicePort")
                }
                this.ingresses.add(new Ingress(name: name, serviceName: backendSvcName, servicePort: backendSvcPort))
            }
            if (obj.kind == 'Deployment') {
                String name = obj.metadata.name
                this.deployments.add(new Deployment(name: name))
            }
        }
        return true
    }
}

class HelmCommandBuilder {
    private StringBuilder sb = new StringBuilder("helm template . ")

    def getCommand() {
        return this.sb.toString()
    }

    def buildIngressCommand(KubeObjects ko) {
        def counter = 0
        for(Ingress ing: ko.getIngresses()) {
            sb.append("--set ingress.${counter}.name=${ing.name} ")
            sb.append("--set ingress.${counter}.serviceName=${ing.serviceName} ")
            sb.append("--set ingress.${counter}.servicePort=${ing.servicePort} ")
            counter++
        }
        return true
    }

    def buildDeploymentCommand(KubeObjects ko) {
        def counter = 0
        for(Deployment deploy: ko.getDeployments()) {
            sb.append("--set deploys.${counter}.name=${deploy.name} ")
            counter++
        }
        return true
    }

    def run(String wd) {
        def proc = sb.toString().execute()
        return proc.consumeProcessOutput()
    }
}

