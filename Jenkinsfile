pipeline {
    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30'))
        disableConcurrentBuilds()
    }

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['PROD', 'DEV', 'UAT'],
            description: 'Target infrastructure environment (PROD default; easily extensible)'
        )
        string(
            name: 'TARGET_SERVERS',
            defaultValue: 'haproxy_servers',
            description: 'Target pattern: "haproxy_servers" for all nodes, or specific host/IP (e.g., 10.0.2.3)'
        )
        string(
            name: 'CERT_DOMAIN',
            defaultValue: 'example.com',
            description: 'Domain name matching Vault path secret/certs/<CERT_DOMAIN>'
        )
        choice(
            name: 'ACTION',
            choices: ['DEPLOY', 'VALIDATE_ONLY'],
            description: 'VALIDATE_ONLY checks cert validity from Vault without modifying servers'
        )
    }

    environment {
        VAULT_ADDR = 'https://192.168.1.209:8200'
        VAULT_SKIP_VERIFY = 'false'
        ANSIBLE_FORCE_COLOR = 'true'
        SSH_CREDENTIAL_ID = 'ansible-ssh-deploy-key'
        VAULT_TOKEN_CRED_ID = "vault-token-${params.ENVIRONMENT.toLowerCase()}"
        TEMP_CERT_DIR = "${WORKSPACE}/.ephemeral_certs"
    }

    stages {
        stage('Initialize & Clean Workspace') {
            steps {
                script {
                    echo "=========================================================="
                    echo "Starting SSL/TLS Automation Pipeline"
                    echo "Environment     : ${params.ENVIRONMENT}"
                    echo "Target Servers  : ${params.TARGET_SERVERS}"
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
                        sh """
                            set +x
                            curl -k -s -S --fail \
                                --header "X-Vault-Token: \${VAULT_TOKEN}" \
                                "${env.VAULT_ADDR}/v1/secret/data/certs/${params.CERT_DOMAIN}" > ${TEMP_CERT_DIR}/vault_response.json || \
                            curl -k -s -S --fail \
                                --header "X-Vault-Token: \${VAULT_TOKEN}" \
                                "${env.VAULT_ADDR}/v1/secret/certs/${params.CERT_DOMAIN}" > ${TEMP_CERT_DIR}/vault_response.json

                            # Extract certificate and private key using python safely
                            python3 -c "
import json, sys

with open('${TEMP_CERT_DIR}/vault_response.json') as f:
    data = json.load(f)

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
                    echo "Executing certificate and private key sanity checks..."
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
                        echo "VALIDATE_ONLY selected. Pipeline finished successfully without modifying servers."
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
                            message: "CONFIRMATION REQUIRED: Deploy SSL certificate for '${params.CERT_DOMAIN}' to PRODUCTION on '${params.TARGET_SERVERS}'?",
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
                    def inventoryFile = "inventories/${params.ENVIRONMENT.toLowerCase()}/hosts.yml"

                    echo "Deploying via Ansible: Inventory=${inventoryFile}, Target=${params.TARGET_SERVERS}"

                    withCredentials([sshUserPrivateKey(credentialsId: env.SSH_CREDENTIAL_ID, keyFileVariable: 'ANSIBLE_SSH_KEY')]) {
                        sh """
                            ansible-playbook -i ${inventoryFile} playbooks/deploy_certs.yml \
                                --limit "${params.TARGET_SERVERS}" \
                                --private-key "\${ANSIBLE_SSH_KEY}" \
                                --extra-vars "cert_domain=${params.CERT_DOMAIN}" \
                                --extra-vars "cert_file_path=${TEMP_CERT_DIR}/cert.crt" \
                                --extra-vars "key_file_path=${TEMP_CERT_DIR}/privkey.key"
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
            echo "SUCCESS: Certificate rotation workflow for ${params.CERT_DOMAIN} on ${params.TARGET_SERVERS} completed successfully!"
        }
        failure {
            echo "FAILURE: Certificate deployment workflow encountered errors. Check logs and rollback status."
        }
    }
}
