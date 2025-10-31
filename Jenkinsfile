pipeline {
    agent {
            kubernetes {
                label 'istio'
                yamlFile "pipeline/pod.yaml"
            }
    }
    parameters {
       string(name: 'namespace', description: 'A namespace yaml should be present tin the namespace-overides folder')
     }
    environment {
        PROJECT_ID = 'meghdo-4567'
        CLUSTER = 'meghdo-cluster'
        REGION = 'europe-west1'
    }
    stages {
        stage('Connect to GKE Cluster') {
            steps {
                script {
                    container('infra-tools') {
                        sh """
                        gcloud container clusters get-credentials ${CLUSTER} --zone ${REGION} --project ${PROJECT_ID}
                        """
                    }
                }
            }
        }
        stage('Namespace and Istio config') {
            steps {
                script {
                    container('infra-tools') {
                      sh """
                         helm upgrade istio-${params.namespace} . \
                                --namespace ${params.namespace} \
                                --create-namespace \
                                --values namespace-overides/${params.namespace}.yaml \
                                --install 
                      """
                    }
                    container('copy-secret') {
                      sh """
                            kubectl label namespace ${params.namespace} istio-injection=enabled
                            kubectl get secret db-password -n default -o yaml | \\
                                grep -v '^\\s*namespace:\\s' | \\
                                grep -v '^\\s*uid:\\s' | \\
                                grep -v '^\\s*resourceVersion:\\s' | \\
                                grep -v '^\\s*creationTimestamp:\\s' | \\
                                kubectl apply -n ${params.namespace} -f -
                      """
                    }
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
