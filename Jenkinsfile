pipeline {
	agent any

	environment {
		// Tweak as needed; Jenkins node should already have sdkmanager installed.
		JAVA_TOOL_OPTIONS = "-Xmx3g"
		GRADLE_OPTS = "-Dorg.gradle.daemon=false -Dorg.gradle.jvmargs='-Xmx3g -Dfile.encoding=UTF-8'"
		// Example: if ANDROID_HOME not set globally:
		// ANDROID_HOME = "/opt/android-sdk"
		// PATH = "${env.ANDROID_HOME}/platform-tools:${env.ANDROID_HOME}/cmdline-tools/latest/bin:${env.PATH}"
	}

	options {
		timestamps()
		ansiColor('xterm')
	}

	stages {
		stage('Checkout') {
			steps {
				checkout scm
			}
		}

		stage('Node & Yarn install') {
			steps {
				sh '''
          node --version || true
          yarn --version || npm i -g yarn
          yarn install --frozen-lockfile
        '''
			}
		}

		stage('Android build (release)') {
			environment {
				// These are Jenkins credentials IDs you create in “Manage Credentials”
				KEYSTORE_ID = 'android_keystore_file' // secret file
				KEYSTORE_PASSWORD = credentials('android_keystore_password')
				KEY_ALIAS = credentials('android_key_alias')
				KEY_PASSWORD = credentials('android_key_password')
			}
			steps {
				dir('android') {
					withCredentials([file(credentialsId: "${env.KEYSTORE_ID}", variable: 'KEYSTORE_PATH')]) {
						sh '''
              # Place keystore where gradle.properties expects it
              cp "$KEYSTORE_PATH" app/app-upload.keystore

              # Optional: warm caches (no-op if already cached on the node)
              mkdir -p $HOME/.gradle/caches
              mkdir -p $HOME/.cache/yarn

              ./gradlew clean :app:assembleRelease -x lint -x test
            '''
					}
				}
			}
		}
	}

	post {
		success {
			archiveArtifacts artifacts: 'android/app/build/outputs/apk/release/*.apk', fingerprint: true
		}
		always {
			junit allowEmptyResults: true, testResults: 'android/**/build/test-results/**/*.xml'
		}
	}
}
