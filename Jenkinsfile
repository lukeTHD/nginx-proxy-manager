pipeline {
	agent any
	options {
		buildDiscarder(logRotator(numToKeepStr: '5'))
		disableConcurrentBuilds()
	}
	environment {
		IMAGE                      = "nginx-proxy-manager"
		BUILD_VERSION              = getVersion()
		MAJOR_VERSION              = "2"
		COMPOSE_FILE               = 'docker/docker-compose.ci.yml'
		COMPOSE_INTERACTIVE_NO_CLI = 1
	}
	stages {
		stage('Environment') {
			parallel {
				stage('Master') {
					when {
						branch 'master'
					}
					steps {
						script {
							env.BUILDX_PUSH_TAGS = "-t docker.io/techizvn/${IMAGE}:${BUILD_VERSION} -t docker.io/techizvn/${IMAGE}:${MAJOR_VERSION} -t docker.io/techizvn/${IMAGE}:latest"
						}
					}
				}
				stage('Versions') {
					steps {
						sh 'cat frontend/package.json | jq --arg BUILD_VERSION "${BUILD_VERSION}" \'.version = $BUILD_VERSION\' | sponge frontend/package.json'
						sh 'echo -e "\\E[1;36mFrontend Version is:\\E[1;33m $(cat frontend/package.json | jq -r .version)\\E[0m"'
						sh 'cat backend/package.json | jq --arg BUILD_VERSION "${BUILD_VERSION}" \'.version = $BUILD_VERSION\' | sponge backend/package.json'
						sh 'echo -e "\\E[1;36mBackend Version is:\\E[1;33m  $(cat backend/package.json | jq -r .version)\\E[0m"'
						sh 'sed -i -E "s/(version-)[0-9]+\\.[0-9]+\\.[0-9]+(-green)/\\1${BUILD_VERSION}\\2/" README.md'
					}
				}
			}
		}
		stage('Frontend') {
			steps {
				sh './scripts/frontend-build'
			}
		}
		stage('Backend') {
			steps {
				echo 'Checking Syntax ...'
				// See: https://github.com/yarnpkg/yarn/issues/3254
				sh '''docker run --rm \\
					-v "$(pwd)/backend:/app" \\
					-v "$(pwd)/global:/app/global" \\
					-w /app \\
					node:latest \\
					sh -c "yarn install && yarn eslint . && rm -rf node_modules"
				'''

				echo 'Docker Build ...'
				sh '''docker build --pull --no-cache --squash --compress \\
					-t "${IMAGE}:ci-${BUILD_NUMBER}" \\
					-f docker/Dockerfile \\
					--build-arg TARGETPLATFORM=linux/amd64 \\
					--build-arg BUILDPLATFORM=linux/amd64 \\
					--build-arg BUILD_VERSION="${BUILD_VERSION}" \\
					--build-arg BUILD_COMMIT="${BUILD_COMMIT}" \\
					--build-arg BUILD_DATE="$(date '+%Y-%m-%d %T %Z')" \\
					.
				'''
			}
		}
	}
}

def getVersion() {
	ver = sh(script: 'cat .version', returnStdout: true)
	return ver.trim()
}

def getCommit() {
	ver = sh(script: 'git log -n 1 --format=%h', returnStdout: true)
	return ver.trim()
}
