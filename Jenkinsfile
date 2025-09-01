pipeline {
    agent { label 'JNode' }

    environment {
        APP_NAME    = "crm"
        RELEASE     = "1.0.0"
        DOCKER_USER = "maheshlokku1999"
        DOCKER_PASS = "docker-token"
        IMAGE_TAG   = "${RELEASE}-${BUILD_NUMBER}"
    }

    tools {
        nodejs 'Node18'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/bytex-analytics/crm.git',
                        credentialsId: 'github-token'
                    ]]
                ])
            }
        }

        stage('Install Frontend Dependencies') {
            steps {
                dir('apps/web') {
                    sh '''
                        rm -rf node_modules package-lock.json
                        npm cache verify
                        npm cache clean --force
                        rm -rf .next dist out build
                        npm install

                        echo " Installing dev dependencies..."
                        npm install tr46@0.0.3 --save-exact
                        npm install --save-dev @types/jsonwebtoken @types/cookie-parser
                        rm -f node_modules/tr46/lib/mappingTable.json

                    '''
                }
            }
        }

        stage('Build Frontend') {
            steps {
                withCredentials([
                    string(credentialsId: 'NEXT_PUBLIC_SUPABASE_URL', variable: 'NEXT_PUBLIC_SUPABASE_URL'),
                    string(credentialsId: 'NEXT_PUBLIC_SUPABASE_ANON_KEY', variable: 'NEXT_PUBLIC_SUPABASE_ANON_KEY'),
                    string(credentialsId: 'NEXTAUTH_SECRET', variable: 'NEXTAUTH_SECRET'),
                    string(credentialsId: 'NEXTAUTH_URL', variable: 'NEXTAUTH_URL'),
                    string(credentialsId: 'EMAIL_USER', variable: 'EMAIL_USER'),
                    string(credentialsId: 'EMAIL_PASSWORD', variable: 'EMAIL_PASSWORD')
                ]) {
                    dir('apps/web') {
                        sh '''
                            echo " Writing environment variables to .env file..."
                            cat <<EOF > .env
                               NEXT_PUBLIC_SUPABASE_URL=$NEXT_PUBLIC_SUPABASE_URL
                               NEXT_PUBLIC_SUPABASE_ANON_KEY=$NEXT_PUBLIC_SUPABASE_ANON_KEY
                               NEXT_PUBLIC_API_URL=http://localhost:4000/graphql
                               NEXTAUTH_SECRET=$NEXTAUTH_SECRET
                               NEXTAUTH_URL=$NEXTAUTH_URL
                               EMAIL_USER=$EMAIL_USER
                               EMAIL_PASSWORD=$EMAIL_PASSWORD
                             EOF

                            echo " Building the frontend..."
                            npm run build
                        '''
                    }
                }
            }
        }

        stage('Build & Push Frontend Image') {
            when {
                expression { currentBuild.resultIsBetterOrEqualTo('SUCCESS') }
            }
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-token') {
                        def feImage = docker.build("${DOCKER_USER}/${APP_NAME}-frontend:${IMAGE_TAG}", "-f apps/web/Dockerfile .")
                        feImage.push()
                        feImage.push("latest")
                    }
                }
            }
        }

        stage('Build & Push Backend Image') {
            when {
                expression { currentBuild.resultIsBetterOrEqualTo('SUCCESS') }
            }
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-token') {
                        def beImage = docker.build("${DOCKER_USER}/${APP_NAME}-backend:${IMAGE_TAG}", "-f apps/api/Dockerfile .")
                        beImage.push()
                        beImage.push("latest")
                    }
                }
            }
        }
    }

    post {
        success {
            echo " Successfully built and pushed frontend & backend images!"
        }
        failure {
            echo " Build failed. Check logs above for error details."
        }
    }
}
