pipeline {
    agent {
            kubernetes {
                label 'istio'
                yamlFile "pipeline/pod.yaml"
            }
    }
    parameters {
       string(name: 'namespace', defaultValue: 'namespace1', description: 'A namespace yaml should be present tin the namespace-overides folder')
     }
    environment {
        CLUSTER = 'pristine-cluster'
        REGION = 'eu-north-1'
    }
    stages {
        stage('Connect to EKS Cluster') {
            steps {
                script {
                    // Use AWS container to connect to EKS cluster
                    container('aws') {
                        sh """
                        aws eks update-kubeconfig --name ${CLUSTER} --region ${REGION}
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
                         helm update --install istio-${params.namespace} \
                                --namespace ${params.namespace} \
                                --create-namespace \
                                --values namespace-overides/${params.namespace}.yaml
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