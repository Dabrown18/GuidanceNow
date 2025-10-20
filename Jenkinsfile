pipeline {
	agent none  // We'll run each stage on its own node label

	triggers { githubPush() }

	environment {
		JAVA_TOOL_OPTIONS = "-Xmx3g"
		GRADLE_OPTS = "-Dorg.gradle.daemon=false -Dorg.gradle.jvmargs='-Xmx3g -Dfile.encoding=UTF-8'"
	}

	options { timestamps() }

	stages {

		stage('Checkout') {
			agent { label 'built-in' }
			steps {
				checkout scm
			}
		}

		stage('Node & Yarn install') {
			agent { label 'built-in' }
			steps {
				retry(3) {
					nodejs(nodeJSInstallationName: 'node20-20.19.5') {
						sh '''
set -xe
node --version
npm --version

yarn --version || npm install -g yarn
yarn --version

yarn install --frozen-lockfile
'''
					}
				}
			}
		}

		stage('Android Release Build') {
			agent { label 'built-in' }
			environment {
				GRADLE_OPTS       = "-Dorg.gradle.daemon=false -Dorg.gradle.jvmargs='-Xmx3g'"
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

cp "$KEYSTORE_PATH" app/app-upload.keystore

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
			agent { label 'mac' }
			environment {
				WORKSPACE = 'GuidanceNow.xcworkspace'
				SCHEME    = 'GuidanceNow'
			}
			steps {
				dir('ios') {
					sh '''
set -xe
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
			node('built-in') {
				echo 'Build succeeded. Archiving APK/AAB and IPA outputs.'
				archiveArtifacts artifacts: 'android/app/build/outputs/apk/release/*.apk', fingerprint: true
				junit allowEmptyResults: true, testResults: 'android/**/build/test-results/**/*.xml'
			}
		}
		failure {
			node('built-in') {
				echo 'Build failed. Please check logs.'
			}
		}
	}
}
