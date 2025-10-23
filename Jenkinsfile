pipeline {
	agent any

	options {
		timestamps()
		// removed ansiColor since the plugin is not installed
	}

	tools {
		// use the NodeJS tool that already exists on this Jenkins
		nodejs 'node20'
	}

	environment {
		ANDROID_HOME = "${env.HOME}/Library/Android/sdk"
		ANDROID_SDK_ROOT = "${env.ANDROID_HOME}"
		JAVA_TOOL_OPTIONS = "-Xmx3g"
		KEYSTORE_PATH = "app-upload.keystore"
		KEYSTORE_TYPE = "PKCS12"
		KEY_ALIAS = "guidanceKey"
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

		stage('Node and deps') {
			steps {
				sh '''
          set -xe
          node --version
          npm --version
          command -v yarn || npm i -g yarn@1
          yarn --version || true

          if [ -f package-lock.json ]; then
            npm ci
          else
            yarn install --frozen-lockfile
          fi
        '''
			}
		}

		stage('Android Release Build + Upload') {
			steps {
				dir('android') {
					withCredentials([
						file(credentialsId: 'android_keystore_file_id', variable: 'KS_FILE'),
						string(credentialsId: 'android_keystore_password_id', variable: 'KS_PASS'),
						string(credentialsId: 'android_key_password_id',     variable: 'KEY_PASS')
					]) {
						sh '''
              set -euo pipefail

              echo "sdk.dir=$ANDROID_HOME" > local.properties
              cat local.properties

              rm -f app/app-upload.keystore
              install -m 600 "$KS_FILE" app/app-upload.keystore
              ls -l app/app-upload.keystore

              echo "Node in PATH: $(which node)"
              node -v
              npm -v

              ./gradlew clean :app:bundleRelease --no-daemon --stacktrace \
                -PMYAPP_UPLOAD_STORE_FILE=app-upload.keystore \
                -PMYAPP_UPLOAD_KEY_ALIAS="$KEY_ALIAS" \
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
			when { expression { return false } }
			steps {
				echo 'iOS step placeholder'
			}
		}
	}
}
