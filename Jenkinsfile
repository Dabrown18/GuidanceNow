pipeline {
	agent { label 'built-in' }  // keep using the controller node

	triggers { githubPush() }

	environment {
		JAVA_TOOL_OPTIONS = "-Xmx3g"
		GRADLE_OPTS = "-Dorg.gradle.daemon=false -Dorg.gradle.jvmargs='-Xmx3g -Dfile.encoding=UTF-8'"

		// Android SDK location (matches your ~/.zshrc)
		ANDROID_SDK_ROOT = "${env.HOME}/Library/Android/sdk"
		ANDROID_HOME     = "${env.ANDROID_SDK_ROOT}"
	}

	options {
		timestamps()
	}

	stages {
		stage('Checkout') {
			steps { checkout scm }
		}

		stage('Node & Yarn install') {
			steps {
				tool name: 'node20', type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation'
				withEnv(["PATH+NODE=${tool name: 'node20', type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation'}/bin"]) {
					sh '''
            set -xe
            node --version
            npm --version
            command -v yarn || npm install -g yarn
            yarn --version || true

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
			when { expression { return isUnix() } }
			steps {
				withCredentials([
					string(credentialsId: 'android_keystore_password', variable: 'KEYSTORE_PASSWORD'),
					string(credentialsId: 'android_key_password',    variable: 'KEY_PASSWORD'),
					string(credentialsId: 'android_key_alias',       variable: 'KEY_ALIAS')
				]) {
					dir('android') {
						withCredentials([file(credentialsId: 'android_keystore_file', variable: 'KEYSTORE_PATH')]) {
							// Ensure Node & Android tools are on PATH if you set them earlier
							withEnv([
								"PATH=${env.ANDROID_HOME ?: '/Users/dbrown/Library/Android/sdk'}/emulator:${env.ANDROID_HOME ?: '/Users/dbrown/Library/Android/sdk'}/platform-tools:${env.PATH}",
								"MYAPP_UPLOAD_STORE_FILE=app/app-upload.keystore"
							]) {
								sh '''
              set -euo pipefail

              chmod +x ./gradlew

              # 1) Clean out any stale keystore in the repo/workspace
              rm -f app/app-upload.keystore

              # 2) Copy the secret-file credential into place (overwrite)
              install -m 600 "$KEYSTORE_PATH" app/app-upload.keystore

              # 3) Create/overwrite local.properties so Gradle finds the SDK
              echo "sdk.dir=/Users/dbrown/Library/Android/sdk" > local.properties
              cat local.properties

              # 4) Sanity-check the keystore BEFORE building
              #    This will fail fast if the secret values don’t match the file.
              keytool -list -v \
                -keystore app/app-upload.keystore -storetype JKS \
                -storepass "$KEYSTORE_PASSWORD" \
                -alias "$KEY_ALIAS" \
                -keypass "$KEY_PASSWORD" >/dev/null

              # 5) Build
              ./gradlew clean :app:bundleRelease --no-daemon
            '''
							}
						}
					}
				}
			}
		}


		stage('iOS Archive + Upload') {
			when { expression { return env.RUN_IOS == 'true' } }
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
