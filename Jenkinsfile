//
// As we are both new to Jenkins, twe went through mutltple iterations to get out file working
// correctly. We learned the code below from the following sources
// [6],[7],[11],[13],[15],[16],[17],[21],[] 
// We also queried GenAI for support with the Hadoop stage. You can find the full transcript 
// in the `Project Sources.txt` file in our github repo.
//
//

pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'SonarScanner-CLI'
        GCP_PROJECT_ID = 'project-option-1-477921'
        GCP_REGION     = 'us-central1'
        GCP_CLUSTER    = 'jenkins-sonarqube-cluster'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/sparks300/python-code-disasters.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Local') {
                    sh "${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=python-code-disasters \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=${SONAR_HOST_URL} \
                        -Dsonar.login=${SONAR_AUTH_TOKEN}"
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                withSonarQubeEnv('SonarQube-Local') {
                    timeout(time: 5, unit: 'MINUTES') { 
                        waitForQualityGate abortPipeline: true
                    }
                }
                echo "SonarQube Quality Gate Passed! Proceeding to Hadoop."
            }
        }
        
        stage('Hadoop MapReduce Job') {
            steps {
                withCredentials([file(credentialsId: 'gcp-service-account-key', variable: 'GCP_KEY_PATH')]) {
                    
                    sh "gcloud auth activate-service-account --key-file=${GCP_KEY_PATH}"
                    

                    sh 'zip -r code_archive.zip . -x "*.git*" -x "node_modules/*" -x "target/*" -x "**/__pycache__/*" -x "code_archive.zip"'

                    script {

                        def stagingBucket = sh(
                            script: "gcloud dataproc clusters describe ${env.GCP_CLUSTER} --region=${env.GCP_REGION} --project=${env.GCP_PROJECT_ID} --format='value(config.configBucket)'",
                            returnStdout: true
                        ).trim()
                        
                        def outputGcsPath = "gs://${stagingBucket}/jenkins-results/line_counts"
                        echo "Dataproc Staging Bucket found: ${stagingBucket}"
                        echo "Job Output Path: ${outputGcsPath}"
                        
                        sh """
                        gcloud dataproc jobs submit pyspark \
                          --cluster=${env.GCP_CLUSTER} \
                          --region=${env.GCP_REGION} \
                          --project=${env.GCP_PROJECT_ID} \
                          --archives=code_archive.zip \
                          line_counter.py -- 'code_archive' '${outputGcsPath}'
                        """
                    }
                }
            }
        }
    }
}