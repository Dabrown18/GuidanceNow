pipeline {
	agent { label 'built-in' }  // keep using the controller node

	// Build on every GitHub push (your webhook will trigger this)
	triggers { githubPush() }

	// Use the NodeJS Plugin tool you configured as "node20"
	tools { nodejs 'node20' }

	environment {
		JAVA_TOOL_OPTIONS = "-Xmx3g"
		GRADLE_OPTS       = "-Dorg.gradle.daemon=false -Dorg.gradle.jvmargs='-Xmx3g -Dfile.encoding=UTF-8'"
	}

	options {
		timestamps()
		// ansiColor('xterm') // leave off unless plugin installed
	}

	stages {
		stage('Checkout') {
			steps { checkout scm }
		}

		stage('Node & Yarn install') {
			steps {
				// Provides PATH to the NodeJS tool named above
				nodejs(nodeJSInstallationName: 'node20') {
					sh '''
            set -xe
            node --version
            npm  --version

            # prefer npm ci if package-lock.json exists, else yarn
            if [ -f package-lock.json ]; then
              npm ci
            else
              command -v yarn || npm install -g yarn
              yarn --version
              yarn install --frozen-lockfile
            fi
          '''
				}
			}
		}

		stage('Android Release Build + Upload') {
			environment {
				KEYSTORE_ID       = 'android_keystore_file'
				KEYSTORE_PASSWORD = credentials('android_keystore_password')
				KEY_ALIAS         = credentials('android_key_alias')
				KEY_PASSWORD      = credentials('android_key_password')
			}
			steps {
				dir('android') {
					withCredentials([file(credentialsId: "${env.KEYSTORE_ID}", variable: 'KEYSTORE_PATH')]) {
						sh '''
              set -xe
              chmod +x ./gradlew || true

              # Put keystore where Gradle expects it
              cp "$KEYSTORE_PATH" app/app-upload.keystore

              # === REQUIRED CHANGE ===
              # Export env vars exactly as build.gradle reads them
              export KEYSTORE_PASSWORD="$KEYSTORE_PASSWORD"
              export KEY_PASSWORD="$KEY_PASSWORD"
              export KEY_ALIAS="$KEY_ALIAS"
              export MYAPP_UPLOAD_STORE_FILE="app/app-upload.keystore"   # <— added

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
              -archivePath build/GuidanceNow.xcarchive clean archive \
              | xcpretty || true

            xcodebuild -exportArchive \
              -archivePath build/GuidanceNow.xcarchive \
              -exportOptionsPlist exportOptions.plist \
              -exportPath build \
              | xcpretty || true
          '''
					withCredentials([
						string(credentialsId: 'ASC_USER',     variable: 'ASC_USER'),
						string(credentialsId: 'ASC_PASSWORD', variable: 'ASC_PASSWORD')
					]) {
						sh '''
              xcrun iTMSTransporter -m upload -assetFile build/GuidanceNow.ipa \
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
