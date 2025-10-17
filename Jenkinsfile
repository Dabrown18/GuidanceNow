pipeline {
	agent any

	// Build on every GitHub push (your webhook will trigger this)
	triggers { githubPush() }

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

		stage('Android Release Build') {
			agent { label 'android' } // or any node with JDK+SDK
			environment {
				GRADLE_OPTS = "-Dorg.gradle.daemon=false -Dorg.gradle.jvmargs='-Xmx3g'"
				KEYSTORE_ID       = 'android_keystore_file'
				KEYSTORE_PASSWORD = credentials('android_keystore_password')
				KEY_ALIAS         = credentials('android_key_alias')
				KEY_PASSWORD      = credentials('android_key_password')
			}
			steps {
				dir('android') {
					// NOTE: removed Play upload creds & task to avoid failures until configured
					withCredentials([file(credentialsId: "${env.KEYSTORE_ID}", variable: 'KEYSTORE_PATH')]) {
						sh '''
              set -e
              chmod +x ./gradlew || true
              cp "$KEYSTORE_PATH" app/app-upload.keystore

              # If your build.gradle reads env vars, export them (harmless if unused):
              export KEYSTORE_PASSWORD="$KEYSTORE_PASSWORD"
              export KEY_PASSWORD="$KEY_PASSWORD"
              export KEY_ALIAS="$KEY_ALIAS"

              ./gradlew clean :app:bundleRelease --no-daemon
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
			// Skip iOS until you’re ready: run with RUN_IOS=true to enable
			when {
				expression { return env.RUN_IOS == 'true' }
			}
			agent { label 'mac' }
			environment {
				WORKSPACE = 'GuidanceNow.xcworkspace'
				SCHEME    = 'GuidanceNow'
			}
			steps {
				dir('ios') {
					sh '''
            set -e
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
					withCredentials([
						string(credentialsId: 'ASC_USER',     variable: 'ASC_USER'),
						string(credentialsId: 'ASC_PASSWORD', variable: 'ASC_PASSWORD')
					]) {
						sh '''
              # Transporter upload (leave as-is; only runs when RUN_IOS=true)
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
