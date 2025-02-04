pipeline {
    agent any // Run the pipeline on any available agent

    // Defining environment variables. I use imaginary variables stored in Jenkins
    environment {
        VENV_DIR            = "venv"
        DOCKER_IMAGE        = credentials('DOCKER_IMAGE') //sample docker placeholder
        SERVER              = credentials('SERVER')
        USER                = credentials('USER')
        APP_DIR             = credentials('APP_DIR')
        ADMIN_DASHBOARD_API = credentials('ADMIN_DASHBOARD_API')
        ADMIN_API_KEY       = credentials('ADMIN_API_KEY')
    }

    stages {
        // 1. Retrieve source code from Git repo linked to Jenkins
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // 2. Setup python venv. Create it if it doesn't exist
        stage('Setup python venv') {
            steps {
                sh '''
                    #create venv if it doesn't exist
                    if [ ! -d "$VENV_DIR" ]; then
                        python3 -m venv $VENV_DIR
                    fi
                    # Update pip, setuptools and wheel for faster installation of dependencies and reduction of errors
                    . $VENV_DIR/bin/activate
                    pip install --upgrade pip setuptools wheel
                '''
            }
        }

        // 3. Install dependencies
        stage('Install dependencies') {
            steps {
                sh '''
                    . $VENV_DIR/bin/activate
                    pip install --no-cache-dir -r requirements.txt
                '''
            }
        }

        // 4. Run migrations for django project
        stage('Run Migrations') {
            steps {
                sh '''
                    . $VENV_DIR/bin/activate
                    python manage.py migrate --noinput
                '''
            }
        }

        // 5. Run django-side test while reusing DB instead of creating one
        stage('Run Tests') {
            steps {
                sh '''
                    . $VENV_DIR/bin/activate
                    python manage.py test --verbosity=2 --keepdb
                '''
            }
        }

        // 6. Lint code using flake8
        stage('Lint code') {
            steps {
                sh '''
                    . $VENV_DIR/bin/activate
                    flake8 TEST
                '''
            }
        }

        // 7. Collect js, css and html from django dirs
        stage('Collect static files') {
            steps {
                sh '''
                    . $VENV_DIR/bin/activate
                    python manage.py collectstatic --noinput
                '''
            }
        }

        // 8. Docker operations
        stage('Build and push Docker image') {
            steps {
                sh '''
                    docker build -t ${DOCKER_IMAGE} .
                    docker push ${DOCKER_IMAGE}
                '''
            }
        }

        // 9. Deployment
        stage('Deploy to Production') {
            //Check if build was successful
            when {
                expression {
                    currentBuild.result == null || currentBuild.result == 'SUCCESS'
                }
            }

            steps {
                sh '''
                    # Django line ensures production ready settings.
                    ssh -o ConnectTimeout=30 ${USER}@${SERVER} << 'EOF'
                        set -e
                        echo "Pulling latest Docker image..."
                        docker pull ${DOCKER_IMAGE}

                        echo "Stopping old container (if running)..."
                        docker stop TEST || true
                        docker rm TEST || true

                        echo "Starting new container..."
                        docker run -d --name TEST --restart always -p 80:8000 \
                            -v ${APP_DIR}/media:/app/media \
                            -v ${APP_DIR}/static:/app/static \
                            -e DJANGO_SETTINGS_MODULE=TEST.settings.production \
                            ${DOCKER_IMAGE}

                        echo "Cleaning up old Docker images..."
                        docker image prune -f
                    EOF
                '''
            }
        }
    }

    // Actions after completing all stages. I added obviously not working sample API calls. Just for fun ¯\_(ツ)_/¯
    post {
    success {
        script {
            def jenkins_logs = sh(script: "tail -n 50 ${env.WORKSPACE}/jenkins.log 2>/dev/null || echo 'No logs found'", returnStdout: true).trim()

            def payload = sh(script: """
                cat <<EOF
                {
                  "status": "success",
                  "job_name": "${env.JOB_NAME}",
                  "build_number": "${env.BUILD_NUMBER}",
                  "timestamp": "$(date -u +"%Y-%m-%dT%H:%M:%SZ")",
                  "jenkins_logs": "$(echo '${jenkins_logs}' | sed ':a;N;$!ba;s/\\n/\\\\n/g' | sed 's/"/\\"/g')"
                }
                EOF
            """, returnStdout: true).trim()

            sh """
                curl -X POST ${ADMIN_DASHBOARD_API} \
                    -H "Content-Type: application/json" \
                    -H "Authorization: Bearer ${ADMIN_API_KEY}" \
                    -d '${payload}'
            """
        }
    }

    failure {
        script {
            def jenkins_logs = sh(script: "tail -n 50 ${env.WORKSPACE}/jenkins.log 2>/dev/null || echo 'No logs found'", returnStdout: true).trim()

            // Sample payload creation including text alteration shenanigans. Meaningless
            def payload = sh(script: """
                cat <<EOF
                {
                  "status": "failure",
                  "job_name": "${env.JOB_NAME}",
                  "build_number": "${env.BUILD_NUMBER}",
                  "timestamp": "$(date -u +"%Y-%m-%dT%H:%M:%SZ")",
                  "error_message": "Pipeline failed. Check Jenkins logs for details.",
                  "jenkins_logs": "$(echo '${jenkins_logs}' | sed ':a;N;$!ba;s/\\n/\\\\n/g' | sed 's/"/\\"/g')"
                }
                EOF
            """, returnStdout: true).trim()

            sh """
                curl -X POST ${ADMIN_DASHBOARD_API} \
                    -H "Content-Type: application/json" \
                    -H "Authorization: Bearer ${ADMIN_API_KEY}" \
                    -d '${payload}'
            """
        }
    }
}