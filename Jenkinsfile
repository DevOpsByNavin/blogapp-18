pipeline {
    agent any

    environment {
        EC2_USER = "ubuntu"
        EC2_HOST = "13.232.45.191"
        EC2_WORKDIR = "blogapp"
    }

    stages {
        stage("Image Tagging") {
            steps {
                script {
                    env.BACKEND1_IMG = "${DH_USERNAME}/blog-backend1:${BUILD_NUMBER}"
                    env.BACKEND1_LATEST = "${DH_USERNAME}/blog-backend1:latest"

                    env.BACKEND2_IMG = "${DH_USERNAME}/blog-backend2:${BUILD_NUMBER}"
                    env.BACKEND2_LATEST = "${DH_USERNAME}/blog-backend2:latest"

                    env.NGINX_IMG = "${DH_USERNAME}/blog-nginx:${BUILD_NUMBER}"
                    env.NGINX_IMG = "${DH_USERNAME}/blog-nginx:latest"
                }
            }
        }

        stage("Build Image") {
            steps {
                withVault( [
                    vaultSecrets: [[
                        path: 'kv/blogapp/frontend'
                        engineVersion: 2
                        secretValues: [
                            [envVar: 'VITE_API_URL', vaultKey: 'VITE_API_URL'],
                            [envVar: 'VITE_CLERK_PUBLISHABLE_KEY', vaultKey: 'VITE_CLERK_PUBLISHABLE_KEY']
                        ]
                    ]],
                    configuration: [
                        vaultUrl: 'https://vault.navin.codes',
                        vaultCredentialId: 'jenkins-blogapp'
                    ]
                ]) {
                sh '''
                    docker build -t ${BACKEND1_IMG} -t ${BACKEND1_LATEST} -f services/backend1/Dockerfile .
                    docker build -t ${BACKEND2_IMG} -t ${BACKEND2_LATEST} -f services/backend2/Dockerfile .

                    cat > services/frontend/.env <<EOF
                    VITE_API_URL: ${VITE_API_URL}
                    VITE_CLERK_PUBLISHABLE_KEY: ${VITE_CLERK_PUBLISHABLE_KEY}
                    EOF

                    docker build -t ${NGINX_IMG} -t ${NGINX_LATEST} -f infra/nginx/Dockerfile .
                '''
                }
            }
        }

        stage("Push image to DockerHub") {
            steps{
                withCredentials([usernamePassword(credentialId: 'dockerhub', usernameVariable: "DH_USER", passwordVariable: "DH_PASSWD")]) {
                    sh '''
                        printf "%s" ${DH_PASSWD} | docker login -u ${DH_USERNAME} --password-stdin

                        docker push ${BACKEND1_IMG}
                        docker push ${BACKEND1_LATEST}

                        docker push ${BACKEND2_IMG}
                        docker push ${BACKEND2_LATEST}

                        docker push ${NGINX_IMG}
                        docker push ${NGINX_LATEST}
                    '''
                }
            }
        }

        stage("Deploy") {
            steps {
                sshagent(credentials: ['ssh-server']) {
                    withVault([
                        configuration: [
                            vaultUrl: 'vault.navin.codes',
                            vaultCredentialId: 'jenkins-blogapp',
                            engineVersion: 2
                        ],
                        vaultSecrets: [
                            [
                                path: 'kv/blogapp/backend1',
                                engineVersion: 2,
                                secretValues: [
                                    [envVar: "CLERK_PUBLISHABLE_KEY", vaultKey: 'CLERK_PUBLISHABLE_KEY'],
                                    [envVar: "CLERK_SECRET_KEY", vaultKey: 'CLERK_SECRET_KEY'],
                                    [envVar: "DB_CLIENT", vaultKey: 'DB_CLIENT'],
                                    [envVar: "DB_HOST", vaultKey: 'DB_HOST'],
                                    [envVar: "DB_NAME", vaultKey: 'DB_NAME'],
                                    [envVar: "DB_PASSWORD", vaultKey: 'DB_PASSWORD'],
                                    [envVar: "DB_PORT", vaultKey: 'DB_PORT'],
                                    [envVar: "DB_USER", vaultKey: 'DB_USER'],
                                    [envVar: "PORT", vaultKey: 'PORT'],
                                ]
                            ],
                            [
                                path: 'kv/blogapp/backend2',
                                engineVersion: 2,
                                secretValues: [
                                    [envVar: "B2_PORT", vaultKey: 'PORT']
                                ]
                            ]
                        ]
                    ]) {
                        sh '''
                            scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -r k8s ${EC2_USER}@${EC2_HOST}:${EC2_WORKDIR}

                            ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${EC2_USER}@${EC2_HOST} "
                            
                                cd ${EC2_WORKDIR}
                                kubectl create configmap backend1-env --from-literal=CLERK_PUBLISHABLE_KEY=${CLERK_PUBLISHABLE_KEY} \
                                --from-literal=CLERK_SECRET_KEY=${CLERK_SECRET_KEY} \
                                --from-literal=DB_CLIENT=${DB_CLIENT} \
                                --from-literal=DB_HOST=${DB_HOST} \
                                --from-literal=DB_NAME=${DB_NAME} \
                                --from-literal=DB_PASSWORD=${DB_PASSWORD} \
                                --from-literal=DB_PORT=${DB_PORT} \
                                --from-literal=DB_USER=${DB_USER} \
                                --from-literal=PORT=${PORT} --dry-run=client -o yaml > k8s/backend1-configmap.yaml

                                kubectl create configmap backend2-env --from-literal=CLERK_PUBLISHABLE_KEY=${CLERK_PUBLISHABLE_KEY} \
                                --from-literal=CLERK_SECRET_KEY=${CLERK_SECRET_KEY} \
                                --from-literal=DB_CLIENT=${DB_CLIENT} \
                                --from-literal=DB_HOST=${DB_HOST} \
                                --from-literal=DB_NAME=${DB_NAME} \
                                --from-literal=DB_PASSWORD=${DB_PASSWORD} \
                                --from-literal=DB_PORT=${DB_PORT} \
                                --from-literal=DB_USER=${DB_USER} \
                                --from-literal=PORT=${B2_PORT} --dry-run=client -o yaml > k8s/backend2-configmap.yaml

                                curl -o init.sql 'https://raw.githubusercontent.com/DevOpsByNavin/blogapp-17/refs/heads/main/infra/database/init.sql'
                                kubectl create configmap postgres-init --from-file init.sql --dry-run=client -o yaml > k8s/postgres-configmap.yaml

                                kubectl apply -f k8s/
                            "
                        '''
                    }
                }
            }
        }
    }
}

