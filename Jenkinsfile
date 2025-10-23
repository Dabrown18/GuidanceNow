pipeline {
	agent any

	options {
		timestamps()
		ansiColor('xterm')
	}

	tools {
		// Configure a NodeJS tool named 'node22' in Jenkins global config first (NodeJS Plugin)
		nodejs 'node22'
	}

	environment {
		// Android SDK location on the agent
		ANDROID_HOME = "${env.HOME}/Library/Android/sdk"
		ANDROID_SDK_ROOT = "${env.ANDROID_HOME}"
		// Gradle memory
		JAVA_TOOL_OPTIONS = "-Xmx3g"
		// App signing envs used by build.gradle
		KEYSTORE_PATH = "app-upload.keystore"       // resolved relative to android/app by Gradle
		KEYSTORE_TYPE = "PKCS12"
		KEY_ALIAS      = "guidanceKey"
		// Node/Yarn will be provided by NodeJS tool above
	}

	stages {

		stage('Checkout') {
			steps {
				checkout([$class: 'GitSCM',
					branches: [[name: '*/main']],
					userRemoteConfigs: [[url: 'https://github.com/Dabrown18/GuidanceNow.git']]
				])
			}
		}

		stage('Node & Yarn install') {
			steps {
				dir('') {
					sh '''
            set -xe
            node --version
            npm --version
            command -v yarn || npm i -g yarn@1
            yarn --version || true

            # prefer package-lock when present, else yarn
            if [ -f package-lock.json ]; then
              npm ci
            else
              yarn install --frozen-lockfile
            fi
          '''
				}
			}
		}

		stage('Android Release Build + Upload') {
			environment {
				// Map Jenkins credentials
				// 1) keystore as a secret file
				// 2) store password and key password must be the same for PKCS12
				// Provide these IDs in Jenkins > Credentials
			}
			steps {
				dir('android') {
					withCredentials([
						file(credentialsId: 'android_keystore_file_id', variable: 'KS_FILE'),
						string(credentialsId: 'android_keystore_password_id', variable: 'KS_PASS'),
						// use the same secret for key password because PKCS12 ignores a different keypass
						string(credentialsId: 'android_key_password_id',     variable: 'KEY_PASS')
					]) {
						sh '''
              set -euo pipefail

              # verify sdk dir
              echo "sdk.dir=$ANDROID_HOME" > local.properties
              cat local.properties

              # place keystore inside android/app where Gradle expects it
              rm -f app/app-upload.keystore
              install -m 600 "$KS_FILE" app/app-upload.keystore
              ls -l app/app-upload.keystore

              echo "Node in PATH: $(which node)"
              node -v
              npm -v

              # Gradle clean + release bundle
              ./gradlew clean :app:bundleRelease --no-daemon --stacktrace \
                -PMYAPP_UPLOAD_STORE_FILE=app-upload.keystore \
                -PMYAPP_UPLOAD_KEY_ALIAS=$KEY_ALIAS \
                -PMYAPP_UPLOAD_STORE_PASSWORD="$KS_PASS" \
                -PMYAPP_UPLOAD_KEY_PASSWORD="$KEY_PASS"
            '''
					}
				}
			}
			post {
				failure {
					archiveArtifacts artifacts: 'android/build/reports/**', allowEmptyArchive: true
				}
				always {
					junit testResults: '**/build/test-results/test/TEST-*.xml', allowEmptyResults: true
				}
			}
		}

		stage('iOS Archive + Upload') {
			when { expression { return false } } // enable later once signing is set up
			steps {
				echo 'iOS step placeholder'
			}
		}
	}
}
