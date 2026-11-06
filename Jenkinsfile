pipeline {
    tools {
        dockerTool 'DEFAULT_DOCKER'
    }
    agent any

    environment {
        UV_HOME               = "/var/jenkins_home/tools/uv"
        UV_INSTALL_DIR        = "${UV_HOME}/bin"
        UV_PYTHON_INSTALL_DIR = "${UV_HOME}/python"
        UV_CACHE_DIR          = "/var/jenkins_home/.cache/uv"
        UV_NO_MODIFY_PATH     = "1"
        UV_INSECURE_HOST      = 'saturn:30088'
        PATH                  = "${UV_HOME}/bin:${PATH}"
        NEXUS_READ            = credentials('nexus-cicd-admin-ro')
        UV_INDEX_SATURN_NEXUS_VIA_INGRESS_USERNAME = "${NEXUS_READ_USR}"
        UV_INDEX_SATURN_NEXUS_VIA_INGRESS_PASSWORD = "${NEXUS_READ_PSW}"
    }

    stages {
        stage('Setup UV (Non-root)') {
            steps {
                sh '''
                    if ! command -v uv >/dev/null 2>&1; then
                        echo "Installing uv into ${UV_INSTALL_DIR} (no root)..."
                        curl -LsSf https://astral.sh/uv/install.sh | sh
                    fi
                    uv --version
                '''
            }
        }

        stage('Sync') {
            steps {
                sh 'uv sync --locked'
                sh 'uv run python --version'
            }
        }

        stage('Lint & Type check') {
            steps {
                sh 'uv run ruff check .'
                sh 'uv run ruff format --check .'
                sh 'uv run mypy src tests'
            }
        }

        stage('Test') {
            steps {
                sh 'uv run pytest -v --junitxml=reports/junit.xml'
            }
            post {
                always { junit allowEmptyResults: true, testResults: 'reports/junit.xml' }
            }
        }

        stage('Build') {
            steps {
                sh 'rm -rf dist'
                sh 'uv build'
                archiveArtifacts artifacts: 'dist/*', fingerprint: true
            }
        }

        stage('Prepare Release') {
            when {
                buildingTag()
            }
            steps {
                script {
                    env.RELEASE_VERSION = env.TAG_NAME.replaceAll(/^v/, '')
                    env.PROJECT_NAME    = sh(script: "uv version | cut -d' ' -f1", returnStdout: true).trim()
                }
                sh '''
                    PROJECT_VERSION=$(uv version --short)
                    if [ "$PROJECT_VERSION" != "$RELEASE_VERSION" ]; then
                        echo "ERROR: tag ($RELEASE_VERSION) != pyproject.toml version ($PROJECT_VERSION)"
                        exit 1
                    fi
                '''
            }
        }

        stage('Generate & Upload SBOM') {
            when {
                buildingTag()
            }
            environment {
                DT_URL = 'http://dependency-track-api-service:8080'
            }
            steps {
                script {
                    echo 'Uploading SBOM to Dependency Track...when it is a tag'
                    withCredentials([string(credentialsId: 'DT_API_KEY', variable: 'DT_API_KEY')]) {
                        sh '''
                            UV_PROJECT_ENVIRONMENT=.venv-sbom uv sync --frozen --no-dev

                            uvx --from cyclonedx-bom cyclonedx-py environment .venv-sbom \
                                --pyproject pyproject.toml --of JSON -o bom.json

                            curl -sSf -X POST "$DT_URL/api/v1/bom" \
                                -H "X-Api-Key: $DT_API_KEY" \
                                -F "autoCreate=true" \
                                -F "projectName=$PROJECT_NAME" \
                                -F "projectVersion=$RELEASE_VERSION" \
                                -F "isLatest=true" \
                                -F "bom=@bom.json"
                        '''
                    }
                    archiveArtifacts artifacts: 'bom.json', fingerprint: true
                }
            }
        }

        stage('Upload to Nexus') {
            when {
                buildingTag()
            }
            environment {
                UV_PUBLISH_URL = 'http://nexus-service:8081/repository/saturn-python-hosted/'
            }
            steps {
                script {
                    echo 'Uploading to Nexus...when it is a tag'
                    withCredentials([usernamePassword(credentialsId: 'nexus-cicd-admin',
                                                      usernameVariable: 'UV_PUBLISH_USERNAME',
                                                      passwordVariable: 'UV_PUBLISH_PASSWORD')]) {
                        sh 'uv publish'
                    }
                }
            }
        }

//         stage('Dockerize (Only on Tag & if Dockerfile exists') {
//             when {
//                 allOf {
//                     buildingTag()
//                     expression { fileExists 'Dockerfile' }
//                 }
//             }
//             steps {
//
//             }
//         }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
    }
}