pipeline {
    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30'))
        disableConcurrentBuilds()
        ansiColor('xterm')
    }

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['UAT', 'PROD'],
            description: 'Target infrastructure environment'
        )
        choice(
            name: 'TARGET_PLATFORM',
            choices: ['ALL', 'KUBERNETES', 'NGINX', 'HAPROXY'],
            description: 'Select infrastructure target to update'
        )
        string(
            name: 'CERT_DOMAIN',
            defaultValue: 'example.com',
            description: 'Domain name matching Vault path secret/certs/<CERT_DOMAIN>'
        )
        choice(
            name: 'ACTION',
            choices: ['DEPLOY', 'VALIDATE_ONLY'],
            description: 'VALIDATE_ONLY tests Vault fetching and cert validity without modifying hosts'
        )
    }

    environment {
        VAULT_ADDR = 'https://vault.internal:8200'
        VAULT_SKIP_VERIFY = 'false'
        ANSIBLE_FORCE_COLOR = 'true'
        SSH_CREDENTIAL_ID = 'ansible-ssh-deploy-key'
        // Tokens configured as Secret Text in Jenkins Credentials Manager
        VAULT_TOKEN_CRED_ID = "${params.ENVIRONMENT == 'PROD' ? 'vault-token-prod' : 'vault-token-uat'}"
        TEMP_CERT_DIR = "${WORKSPACE}/.ephemeral_certs"
    }

    stages {
        stage('Initialize & Clean Workspace') {
            steps {
                script {
                    echo "=========================================================="
                    echo "Starting SSL/TLS Automation Pipeline"
                    echo "Environment     : ${params.ENVIRONMENT}"
                    echo "Target Platform : ${params.TARGET_PLATFORM}"
                    echo "Cert Domain     : ${params.CERT_DOMAIN}"
                    echo "Action          : ${params.ACTION}"
                    echo "=========================================================="

                    sh "rm -rf ${TEMP_CERT_DIR}"
                    sh "mkdir -p -m 0700 ${TEMP_CERT_DIR}"
                }
            }
        }

        stage('Fetch Certificate from HashiCorp Vault') {
            steps {
                withCredentials([string(credentialsId: env.VAULT_TOKEN_CRED_ID, variable: 'VAULT_TOKEN')]) {
                    script {
                        echo "Connecting to Vault at ${env.VAULT_ADDR}..."
                        // Fetching secrets via Vault HTTP API (supports both KV v2 and v1)
                        sh """
                            set +x
                            curl -s -S --fail \
                                --header "X-Vault-Token: \${VAULT_TOKEN}" \
                                "${env.VAULT_ADDR}/v1/secret/data/certs/${params.CERT_DOMAIN}" > ${TEMP_CERT_DIR}/vault_response.json || \
                            curl -s -S --fail \
                                --header "X-Vault-Token: \${VAULT_TOKEN}" \
                                "${env.VAULT_ADDR}/v1/secret/certs/${params.CERT_DOMAIN}" > ${TEMP_CERT_DIR}/vault_response.json

                            # Extract certificate and private key using python/jq safely
                            python3 -c "
import json, sys

with open('${TEMP_CERT_DIR}/vault_response.json') as f:
    data = json.load(f)

# Handle KV v2 ('data' -> 'data') or KV v1 ('data')
secret_data = data.get('data', {}).get('data', data.get('data', {}))

cert = secret_data.get('tls.crt') or secret_data.get('certificate') or secret_data.get('cert')
key = secret_data.get('tls.key') or secret_data.get('private_key') or secret_data.get('key')

if not cert or not key:
    print('ERROR: Could not find cert or key in Vault payload!', file=sys.stderr)
    sys.exit(1)

with open('${TEMP_CERT_DIR}/cert.crt', 'w') as fc:
    fc.write(cert.strip() + '\\n')

with open('${TEMP_CERT_DIR}/privkey.key', 'w') as fk:
    fk.write(key.strip() + '\\n')
"
                            chmod 600 ${TEMP_CERT_DIR}/privkey.key
                            chmod 644 ${TEMP_CERT_DIR}/cert.crt
                            rm -f ${TEMP_CERT_DIR}/vault_response.json
                        """
                    }
                }
            }
        }

        stage('Pre-flight Validation') {
            steps {
                script {
                    echo "Executing basic certificate and private key sanity checks..."
                    sh """
                        # 1. Check files are non-empty
                        if [ ! -s ${TEMP_CERT_DIR}/cert.crt ]; then
                            echo "ERROR: cert.crt is empty!" >&2; exit 1
                        fi
                        if [ ! -s ${TEMP_CERT_DIR}/privkey.key ]; then
                            echo "ERROR: privkey.key is empty!" >&2; exit 1
                        fi

                        # 2. Check OpenSSL readability & validity
                        echo "--- Certificate Metadata ---"
                        openssl x509 -in ${TEMP_CERT_DIR}/cert.crt -noout -subject -dates -issuer

                        echo "--- Private Key Integrity ---"
                        openssl pkey -in ${TEMP_CERT_DIR}/privkey.key -noout || openssl rsa -in ${TEMP_CERT_DIR}/privkey.key -check -noout

                        echo "Pre-flight validation passed successfully!"
                    """

                    if (params.ACTION == 'VALIDATE_ONLY') {
                        echo "VALIDATE_ONLY selected. Pipeline finished successfully without applying changes."
                    }
                }
            }
        }

        stage('Production Approval Gate') {
            when {
                allOf {
                    expression { return params.ENVIRONMENT == 'PROD' }
                    expression { return params.ACTION == 'DEPLOY' }
                }
            }
            steps {
                script {
                    timeout(time: 1, unit: 'HOURS') {
                        input(
                            id: 'ProceedWithProdDeploy',
                            message: "CONFIRMATION REQUIRED: Deploy SSL certificate for '${params.CERT_DOMAIN}' to PRODUCTION (${params.TARGET_PLATFORM})?",
                            ok: 'Approve & Deploy to PROD'
                        )
                    }
                }
            }
        }

        stage('Execute Ansible Certificate Deployment') {
            when {
                expression { return params.ACTION == 'DEPLOY' }
            }
            steps {
                script {
                    def inventoryFile = (params.ENVIRONMENT == 'PROD') ? 'inventories/prod/hosts.yml' : 'inventories/uat/hosts.yml'
                    def ansibleTags = ""

                    switch(params.TARGET_PLATFORM) {
                        case 'KUBERNETES':
                            ansibleTags = "kubernetes"
                            break
                        case 'NGINX':
                            ansibleTags = "nginx,verify"
                            break
                        case 'HAPROXY':
                            ansibleTags = "haproxy,verify"
                            break
                        default:
                            ansibleTags = "all"
                            break
                    }

                    echo "Deploying via Ansible: Inventory=${inventoryFile}, Tags=${ansibleTags}"

                    sshagent([env.SSH_CREDENTIAL_ID]) {
                        // Pass cert content safely without leaking private key into build log
                        sh """
                            set +x
                            ansible-playbook -i ${inventoryFile} playbooks/deploy_certs.yml \
                                --tags "${ansibleTags}" \
                                --extra-vars "cert_domain=${params.CERT_DOMAIN}" \
                                --extra-vars "cert_content=\\"$(cat ${TEMP_CERT_DIR}/cert.crt)\\"" \
                                --extra-vars "key_content=\\"$(cat ${TEMP_CERT_DIR}/privkey.key)\\""
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                echo "ZERO TRUST HYGIENE: Shredding and purging ephemeral certificate files from workspace..."
                sh """
                    if [ -d "${TEMP_CERT_DIR}" ]; then
                        find "${TEMP_CERT_DIR}" -type f -exec shred -u -z -n 3 {} + 2>/dev/null || rm -rf "${TEMP_CERT_DIR}"
                    fi
                """
            }
        }
        success {
            echo "SUCCESS: Certificate rotation workflow for ${params.CERT_DOMAIN} completed successfully!"
        }
        failure {
            echo "FAILURE: Certificate deployment workflow encountered errors. Check logs and rollback status."
        }
    }
}
