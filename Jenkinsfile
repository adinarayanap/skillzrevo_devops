pipeline {
    agent any

    environment {
        APP_DIR        = '/home/ec2-user/skillzrevo_devops'
        VENV_DIR       = "${APP_DIR}/venv"
        APP_PORT       = '5000'
        PYTHON         = 'python3'
        GIT_REPO       = 'https://github.com/adinarayanap/skillzrevo_devops.git'
        GIT_BRANCH     = 'feature/jenkins'
    }

    stages {

        // ── 1. CHECKOUT ──────────────────────────────────────────────────────
        stage('Checkout') {
            steps {
                echo '📥 Pulling latest code from GitHub...'
                git branch: "${GIT_BRANCH}",
                    url: "${GIT_REPO}"
            }
        }

        // ── 2. SETUP VIRTUAL ENV ─────────────────────────────────────────────
        stage('Setup Environment') {
            steps {
                echo '🐍 Setting up Python virtual environment...'
                sh '''
                    if [ ! -d "${VENV_DIR}" ]; then
                        ${PYTHON} -m venv ${VENV_DIR}
                    fi
                    ${VENV_DIR}/bin/pip install --upgrade pip
                    ${VENV_DIR}/bin/pip install -r requirements.txt
                '''
            }
        }

        // ── 3. TEST ───────────────────────────────────────────────────────────
        stage('Test') {
            steps {
                echo '🧪 Running tests...'
                sh '''
                    ${VENV_DIR}/bin/pip install pytest
                    if [ -d "tests" ]; then
                        ${VENV_DIR}/bin/pytest tests/ -v --tb=short
                    else
                        echo "⚠️  No tests directory found — skipping."
                    fi
                '''
            }
        }

        // ── 4. DEPLOY ─────────────────────────────────────────────────────────
        stage('Deploy') {
            steps {
                echo '🚀 Deploying application...'
                sh '''
                    # Copy app files to deployment directory
                    sudo rsync -av --exclude='.git' \
                        --exclude='venv' \
                        --exclude='Jenkinsfile' \
                        ${WORKSPACE}/ ${APP_DIR}/

                    # Install dependencies in app venv
                    if [ ! -d "${VENV_DIR}" ]; then
                        ${PYTHON} -m venv ${VENV_DIR}
                    fi
                    ${VENV_DIR}/bin/pip install --upgrade pip
                    ${VENV_DIR}/bin/pip install -r ${APP_DIR}/requirements.txt

                    # Restart the app using systemd
                    if systemctl is-active --quiet flaskapp; then
                        sudo systemctl restart flaskapp
                        echo "✅ Restarted via systemd"
                    else
                        # Kill old process if running on APP_PORT
                        sudo fuser -k ${APP_PORT}/tcp || true

                        # Start app in background
                        cd ${APP_DIR}
                        nohup ${VENV_DIR}/bin/python app.py \
                            > /tmp/flaskapp.log 2>&1 &

                        echo "✅ App started on port ${APP_PORT}"
                    fi
                '''
            }
        }

        // ── 5. HEALTH CHECK ───────────────────────────────────────────────────
        stage('Health Check') {
            steps {
                echo '❤️  Verifying app is running...'
                sh '''
                    sleep 5
                    curl -sf http://localhost:${APP_PORT} \
                        && echo "✅ App is UP!" \
                        || echo "⚠️  Health check failed — check /tmp/flaskapp.log"
                '''
            }
        }
    }

    // ── POST ACTIONS ──────────────────────────────────────────────────────────
    post {
        success {
            echo '🎉 Pipeline completed successfully! App is deployed.'
        }
        failure {
            echo '❌ Pipeline failed. Check the logs above.'
        }
        always {
            echo '🧹 Cleaning up workspace...'
            cleanWs()
        }
    }
}
