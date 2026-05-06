pipeline {
    agent any

    tools {
        maven 'Maven 3.9'
        jdk   'JDK-17'
    }

    environment {
        GCP_PROJECT  = 'your-gcp-project-id'     
        APP_NAME     = 'sync-service'
        SERVICE_PORT = '8080'
        SONAR_ORG         = 'your-sonarcloud-org'    
        SONAR_PROJECT_KEY = 'your-org_sync-service' 

        // 'gcp-sa-key' → Jenkins "Secret file" credential containing the GCP
        // service account JSON key. 
        // this variable to that path. gcloud picks it up automatically.
        GOOGLE_APPLICATION_CREDENTIALS = credentials('gcp-sa-key')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    def sha = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.ARTIFACT_NAME = "${env.APP_NAME}-${sha}-${env.BUILD_NUMBER}.jar"
                    env.DEPLOY_ENV    = resolveEnv(env.BRANCH_NAME ?: '')
                    echo "Branch: ${env.BRANCH_NAME} → target env: ${env.DEPLOY_ENV}"
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests --batch-mode --no-transfer-progress'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test jacoco:report --batch-mode --no-transfer-progress'
            }
            post {
                always { junit 'target/surefire-reports/*.xml' }
            }
        }
        stage('SonarCloud Analysis') {
            steps {
                withSonarQubeEnv('SonarCloud') {
                    sh """
                        mvn sonar:sonar \
                            -Dsonar.projectKey=${env.SONAR_PROJECT_KEY} \
                            -Dsonar.organization=${env.SONAR_ORG} \
                            -Dsonar.branch.name=${env.BRANCH_NAME} \
                            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                            -Dsonar.java.binaries=target/classes \
                            --batch-mode --no-transfer-progress
                    """
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ── feature/* and hotfix/* (PR builds) exit here ─────────────────────
        // No artifact is uploaded.
        // develop / release/* / main continue below.

        stage('Upload Artifact') {
            when { expression { env.DEPLOY_ENV != 'none' } }
            steps {
                sh "cp target/${env.APP_NAME}.jar ${env.ARTIFACT_NAME}"

                // Google Cloud Storage Plugin for Jenkins.
                googleStorageUpload(
                    credentialsId: 'gcp-gcs-credentials',
                    bucket: "gs://${env.GCS_BUCKET}/${env.DEPLOY_ENV}/",
                    pattern: env.ARTIFACT_NAME
                )
            }
        }

        // Only members of the 'release-managers' Jenkins group can proceed.
        // Times out after 30 min.
        stage('Approve Prod Deploy') {
            when { branch 'main' }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input(
                        message: "Deploy ${env.ARTIFACT_NAME} to PRODUCTION?",
                        ok: 'Deploy',
                        submitter: 'release-managers'
                    )
                }
            }
        }

        stage('Deploy') {
            when { expression { env.DEPLOY_ENV != 'none' } }
            steps {
                script {
                    sh "gcloud auth activate-service-account --key-file=${env.GOOGLE_APPLICATION_CREDENTIALS} --project=${env.GCP_PROJECT}"

                    // Snapshot what is currently running so rollback has a restore target
                    try {
                        googleStorageDownload(
                            credentialsId: 'gcp-gcs-credentials',
                            bucketUri: "gs://${env.GCS_BUCKET}/${env.DEPLOY_ENV}/current-version.txt",
                            localDirectory: '.',
                            pathPrefix: "${env.DEPLOY_ENV}/"
                        )
                        env.PREVIOUS_VERSION = readFile('current-version.txt').trim()
                    } catch (e) {
                        env.PREVIOUS_VERSION = 'none'
                    }
                    echo "Previous version: ${env.PREVIOUS_VERSION}"

                    if (env.DEPLOY_ENV == 'prod') {
                        deployBlueGreen()
                    } else {
                        // qa  → 1 VM  → effectively Recreate
                        // staging → 2 VMs → Rolling (one at a time with health check between)
                        def vms = getVMsByTag("${env.APP_NAME}-${env.DEPLOY_ENV}")
                        deployRolling(vms)
                    }

                    // Commit point — only reached if deploy + health checks succeeded
                    writeFile file: 'current-version.txt', text: env.ARTIFACT_NAME
                    googleStorageUpload(
                        credentialsId: 'gcp-gcs-credentials',
                        bucket: "gs://${env.GCS_BUCKET}/${env.DEPLOY_ENV}/",
                        pattern: 'current-version.txt'
                    )
                }
            }
        }

        stage('Health Check') {
            when { expression { env.DEPLOY_ENV != 'none' } }
            steps {
                script {
                    def vms = getVMsByTag("${env.APP_NAME}-${env.DEPLOY_ENV}")
                    vms.each { vm -> healthCheck(vm.name, vm.zone) }

                    // record this artifact as the current stable version.
                    // The rollback block reads this file to know what to restore.
                    sh "printf '${env.ARTIFACT_NAME}' | gsutil cp - gs://${env.GCS_BUCKET}/${env.DEPLOY_ENV}/current-version.txt"
                }
            }
        }
    }

    post {
        failure {
            script {
                if (env.DEPLOY_ENV && env.DEPLOY_ENV != 'none'
                        && env.PREVIOUS_VERSION && env.PREVIOUS_VERSION != 'none') {
                    echo "Initiating rollback to ${env.PREVIOUS_VERSION}..."
                    try {
                        sh "gcloud auth activate-service-account --key-file=${env.GOOGLE_APPLICATION_CREDENTIALS} --project=${env.GCP_PROJECT}"

                        if (env.DEPLOY_ENV == 'prod' && env.LIVE_COLOR) {
                            // Blue/Green rollback — flip LB back to the previously live color.
                            // The live-color VMs never stopped; this takes effect in seconds.
                            sh """
                                gcloud compute backend-services update ${env.APP_NAME}-prod-backend \
                                    --global \
                                    --project=${env.GCP_PROJECT} \
                                    --update-backends="name=${env.APP_NAME}-prod-${env.LIVE_COLOR},balancing-mode=UTILIZATION,max-utilization=0.8,capacity-scaler=1.0" \
                                    --update-backends="name=${env.APP_NAME}-prod-${env.DEPLOY_COLOR},balancing-mode=UTILIZATION,max-utilization=0.8,capacity-scaler=0.0"
                            """
                            echo "Prod LB rolled back to ${env.LIVE_COLOR}"
                        } else {
                            // Rolling rollback — re-deploy previous JAR to each VM
                            def vms = getVMsByTag("${env.APP_NAME}-${env.DEPLOY_ENV}")
                            vms.each { vm -> rollbackVM(vm.name, vm.zone, env.PREVIOUS_VERSION) }
                        }
                    } catch (e) {
                        echo "Rollback failed — manual intervention required: ${e.message}"
                    }
                }
            }
        }
        always { cleanWs() }
    }
}


// =============================================================================
// Helper functions
// =============================================================================

// Maps branch name to a deployment environment.
// Anything not matched returns 'none' (PR / feature / hotfix builds).
def resolveEnv(String branch) {
    if (branch == 'develop')           return 'qa'
    if (branch ==~ /^release\/.+/)     return 'staging'
    if (branch == 'main')              return 'prod'
    return 'none'
}

// Returns a list of maps [{name, zone}] for all RUNNING VMs carrying the given tag.
def getVMsByTag(String tag) {
    def raw = sh(
        script: """
            gcloud compute instances list \
                --filter="tags.items=${tag} AND status=RUNNING" \
                --format="csv[no-heading](name,zone)" \
                --project=${env.GCP_PROJECT}
        """,
        returnStdout: true
    ).trim()

    if (!raw) error("No running VMs found with tag '${tag}'. Check the tag and GCP project.")

    return raw.split('\n').collect { line ->
        def parts = line.trim().split(',')
        [name: parts[0], zone: parts[1]]
    }
}

// Rolling deploy — used for QA and Staging.
// Each VM is deployed and verified individually before moving to the next.

def deployRolling(List vms) {
    vms.each { vm ->
        echo "Rolling deploy → ${vm.name}"
        deployToVM(vm.name, vm.zone)
        // Verify this VM before touching the next one.
        // If health check fails here, the pipeline fails and rollback runs —
        // the remaining VMs are still on the old version (safe state).
        healthCheck(vm.name, vm.zone)
        echo "${vm.name} healthy — continuing roll"
    }
}
// -----------------------------------------------------------------------------
// Blue/Green deploy — used for Prod only.
//
// VM tag convention:
//   sync-service-prod-blue   → blue instance group
//   sync-service-prod-green  → green instance group
//
// GCS tracks which color is live.
// -----------------------------------------------------------------------------
def deployBlueGreen() {
    // Determine which color is currently live; deploy to the idle one
    def liveColor = 'blue'
    try {
        googleStorageDownload(
            credentialsId: 'gcp-gcs-credentials',
            bucketUri: "gs://${env.GCS_BUCKET}/prod/live-color.txt",
            localDirectory: '.',
            pathPrefix: 'prod/'
        )
        liveColor = readFile('live-color.txt').trim()
    } catch (e) {
        echo "live-color.txt not found — first prod deploy, defaulting live=blue"
    }

    def deployColor = (liveColor == 'blue') ? 'green' : 'blue'
    env.LIVE_COLOR   = liveColor
    env.DEPLOY_COLOR = deployColor
    echo "Blue/Green: live=${liveColor} → deploying to=${deployColor}"

    // Deploy to all idle-color VMs
    def idleVMs = getVMsByTag("${env.APP_NAME}-prod-${deployColor}")
    idleVMs.each { vm -> deployToVM(vm.name, vm.zone) }

    // Health-check idle VMs directly
    idleVMs.each { vm -> healthCheck(vm.name, vm.zone) }

    sh """
        gcloud compute backend-services update ${env.APP_NAME}-prod-backend \
            --global \
            --project=${env.GCP_PROJECT} \
            --update-backends="name=${env.APP_NAME}-prod-${deployColor},balancing-mode=UTILIZATION,max-utilization=0.8,capacity-scaler=1.0" \
            --update-backends="name=${env.APP_NAME}-prod-${liveColor},balancing-mode=UTILIZATION,max-utilization=0.8,capacity-scaler=0.0"
    """
    echo "Traffic switched: ${liveColor} → ${deployColor}"

    // Persist new live color so the next deploy knows which side to target
    writeFile file: 'live-color.txt', text: deployColor
    googleStorageUpload(
        credentialsId: 'gcp-gcs-credentials',
        bucket: "gs://${env.GCS_BUCKET}/prod/",
        pattern: 'live-color.txt'
    )
}

// Writes a deploy script locally, scps it to the VM, and executes it.
// Using a script file sidesteps all shell-quoting complexity for multi-step
// remote commands that themselves contain variable expansions.
def deployToVM(String vm, String zone) {
    writeFile file: 'deploy.sh', text: """#!/bin/bash
set -euo pipefail

# Download artifact from GCS using the instance-attached service account.
# 'gcloud storage cp' is the current recommended replacement for 'gsutil cp'.
gcloud storage cp \\
    gs://${env.GCS_BUCKET}/${env.DEPLOY_ENV}/${env.ARTIFACT_NAME} \\
    /tmp/${env.ARTIFACT_NAME}

sudo mv /tmp/${env.ARTIFACT_NAME} /opt/${env.APP_NAME}/${env.APP_NAME}.jar
sudo chown ${env.APP_NAME}:${env.APP_NAME} /opt/${env.APP_NAME}/${env.APP_NAME}.jar

# Fetch the MongoDB URI from Secret Manager on the VM itself.
# The secret value is written directly to the env file and never echoed to logs.
MONGODB_URI=\$(gcloud secrets versions access latest \\
    --secret=${env.APP_NAME}-${env.DEPLOY_ENV}-mongodb-uri \\
    --project=${env.GCP_PROJECT})

printf 'MONGODB_URI=%s\\n' "\$MONGODB_URI" | sudo tee /etc/sync-service/secrets.env > /dev/null
sudo chmod 600 /etc/sync-service/secrets.env
sudo chown ${env.APP_NAME}:${env.APP_NAME} /etc/sync-service/secrets.env

sudo systemctl restart ${env.APP_NAME}
"""

    sh """
        gcloud compute scp deploy.sh ${vm}:/tmp/deploy.sh \
            --zone=${zone} --project=${env.GCP_PROJECT}
        gcloud compute ssh ${vm} --zone=${zone} --project=${env.GCP_PROJECT} \
            --command='bash /tmp/deploy.sh && rm -f /tmp/deploy.sh'
    """
}

// Re-deploys the last known good artifact.
// For prod Blue/Green, extend this to also flip the LB backend back.
def rollbackVM(String vm, String zone, String artifact) {
    writeFile file: 'rollback.sh', text: """#!/bin/bash
set -euo pipefail
gcloud storage cp \\
    gs://${env.GCS_BUCKET}/${env.DEPLOY_ENV}/${artifact} \\
    /tmp/${artifact}
sudo mv /tmp/${artifact} /opt/${env.APP_NAME}/${env.APP_NAME}.jar
sudo chown ${env.APP_NAME}:${env.APP_NAME} /opt/${env.APP_NAME}/${env.APP_NAME}.jar
sudo systemctl restart ${env.APP_NAME}
"""

    sh """
        gcloud compute scp rollback.sh ${vm}:/tmp/rollback.sh \
            --zone=${zone} --project=${env.GCP_PROJECT}
        gcloud compute ssh ${vm} --zone=${zone} --project=${env.GCP_PROJECT} \
            --command='bash /tmp/rollback.sh && rm -f /tmp/rollback.sh'
    """
}

// Polls /actuator/health every 5 s for up to 90 s.
// Runs on VM only.
def healthCheck(String vm, String zone) {
    writeFile file: 'healthcheck.sh', text: """#!/bin/bash
for i in \$(seq 1 18); do
    STATUS=\$(curl -s http://localhost:${env.SERVICE_PORT}/actuator/health 2>/dev/null \\
        | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])" 2>/dev/null \\
        || echo DOWN)
    [ "\$STATUS" = "UP" ] && echo "Health check passed." && exit 0
    echo "Attempt \$i/18: \$STATUS — retrying in 5s"
    sleep 5
done
echo "Health check timed out after 90s."
exit 1
"""

    sh """
        gcloud compute scp healthcheck.sh ${vm}:/tmp/healthcheck.sh \
            --zone=${zone} --project=${env.GCP_PROJECT}
        gcloud compute ssh ${vm} --zone=${zone} --project=${env.GCP_PROJECT} \
            --command='bash /tmp/healthcheck.sh && rm -f /tmp/healthcheck.sh'
    """
}



