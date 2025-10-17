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

		stage('Android Release Build + Upload') {
			agent {
				label 'android'
			} // or any node with JDK+SDK
			environment {
				GRADLE_OPTS = "-Dorg.gradle.daemon=false -Dorg.gradle.jvmargs='-Xmx3g'"
				KEYSTORE_ID = 'android_keystore_file'
				KEYSTORE_PASSWORD = credentials('android_keystore_password')
				KEY_ALIAS = credentials('android_key_alias')
				KEY_PASSWORD = credentials('android_key_password')
			}
			steps {
				dir('android') {
					withCredentials([file(credentialsId: "${env.KEYSTORE_ID}", variable: 'KEYSTORE_PATH'),
						file(credentialsId: 'play_service_json', variable: 'PLAY_JSON_PATH')]) {
						sh '''
          cp "$KEYSTORE_PATH" app/app-upload.keystore
          export PLAY_SERVICE_JSON="$PLAY_JSON_PATH"
          ./gradlew clean :app:bundleRelease :app:publishRelease --no-daemon
        '''
					}
				}
			}
			post {
				success {
					archiveArtifacts artifacts: 'android/app/build/outputs/**/*.aab', fingerprint: true
				}
			}
		}

		stage('iOS Archive + Upload') {
			agent {
				label 'mac'
			}
			environment {
				WORKSPACE = 'GuidanceNow.xcworkspace'
				SCHEME = 'GuidanceNow'
			}
			steps {
				dir('ios') {
					sh '''
        gem install bundler --no-document || true
        bundle install || true
        bundle exec pod install --repo-update
        xcodebuild -workspace "$WORKSPACE" -scheme "$SCHEME" -configuration Release \
          -archivePath build/YourApp.xcarchive clean archive \
          | xcpretty || true
        xcodebuild -exportArchive \
          -archivePath build/YourApp.xcarchive \
          -exportOptionsPlist exportOptions.plist \
          -exportPath build \
          | xcpretty || true
      '''
					// Upload via Transporter using Apple ID app-specific password
					withCredentials([string(credentialsId: 'ASC_USER', variable: 'ASC_USER'),
						string(credentialsId: 'ASC_PASSWORD', variable: 'ASC_PASSWORD')]) {
						sh '''
          xcrun iTMSTransporter -m upload -assetFile build/YourApp.ipa \
            -u "$ASC_USER" -p "$ASC_PASSWORD" -itc_provider YOUR_TEAM_ID
        '''
					}
				}
			}
			post {
				success {
					archiveArtifacts artifacts: 'ios/build/**/*.ipa', fingerprint: true
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
