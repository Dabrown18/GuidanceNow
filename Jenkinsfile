pipeline {
	agent { label 'built-in' }

	triggers { githubPush() }

	environment {
		JAVA_TOOL_OPTIONS = "-Xmx3g"
		GRADLE_OPTS = "-Dorg.gradle.daemon=false -Dorg.gradle.jvmargs='-Xmx3g -Dfile.encoding=UTF-8'"
		ANDROID_SDK_ROOT = "${env.HOME}/Library/Android/sdk"
		ANDROID_HOME     = "${env.ANDROID_SDK_ROOT}"
	}

	options { timestamps() }

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
							withEnv([
								"PATH=${tool(name: 'node20', type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation')}/bin:${env.ANDROID_HOME ?: '/Users/dbrown/Library/Android/sdk'}/emulator:${env.ANDROID_HOME ?: '/Users/dbrown/Library/Android/sdk'}/platform-tools:${env.PATH}",
								"MYAPP_UPLOAD_STORE_FILE=app/app-upload.keystore"
							]) {
								sh '''
                  set -euo pipefail

                  chmod +x ./gradlew
                  rm -f app/app-upload.keystore
                  install -m 600 "$KEYSTORE_PATH" app/app-upload.keystore

                  echo "sdk.dir=${ANDROID_HOME:-/Users/dbrown/Library/Android/sdk}" > local.properties
                  cat local.properties

                  keytool -list -v \
                    -keystore app/app-upload.keystore -storetype JKS \
                    -storepass "$KEYSTORE_PASSWORD" \
                    -alias "$KEY_ALIAS" \
                    -keypass "$KEY_PASSWORD" >/dev/null

                  echo "Node in Gradle PATH: $(which node)"
                  node -v
                  npm -v

                  ./gradlew clean :app:bundleRelease --no-daemon
                '''
							}
						}
					}
				}
			}
		}

		stage('iOS Archive + Upload') {
			environment {
				WORKSPACE    = 'GuidanceNow.xcworkspace'
				SCHEME       = 'GuidanceNow'
				CONFIG       = 'Release'
				ARCHIVE_PATH = 'build/GuidanceNow.xcarchive'
				EXPORT_DIR   = 'build'
				EXPORT_PLIST = 'exportOptions.plist'
			}
			steps {
				dir('ios') {
					sh '''
            set -euo pipefail
            export LANG=en_US.UTF-8
            export LC_ALL=en_US.UTF-8

            # Use user-local gem path (no sudo/system writes)
            export GEM_HOME="$HOME/.gem"
            export PATH="$GEM_HOME/bin:$PATH"

            which xcodebuild
            xcodebuild -version || true
            ruby -v
            gem -v

            # Match Bundler to Gemfile.lock; fallback to 2.5.11 if not present
            BUNDLER_VER="$(ruby -e 'f=\"Gemfile.lock\"; v=/BUNDLED WITH\\n\\s+([0-9.]+)/.match(File.read(f)) rescue nil; puts(v ? v[1] : \"2.5.11\")')"
            gem list -i bundler -v "$BUNDLER_VER" >/dev/null || gem install --user-install bundler:"$BUNDLER_VER" --no-document

            # If CocoaPods isn't in Gemfile, install user-local
            if ! grep -qi cocoapods Gemfile 2>/dev/null; then
              gem list -i cocoapods >/dev/null || gem install --user-install cocoapods --no-document
            fi

            # Install gems into project-local vendor/bundle
            bundle _${BUNDLER_VER}_ config set --local path 'vendor/bundle'
            bundle _${BUNDLER_VER}_ install --jobs=4

            # Pods
            bundle _${BUNDLER_VER}_ exec pod install --repo-update

            # Ensure exportOptions.plist exists (adjust method if needed)
            if [ ! -f "$EXPORT_PLIST" ]; then
              cat > "$EXPORT_PLIST" <<'PLIST'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key><string>ad-hoc</string>
  <key>compileBitcode</key><false/>
  <key>stripSwiftSymbols</key><true/>
  <key>signingStyle</key><string>automatic</string>
  <key>destination</key><string>export</string>
  <key>manageAppVersionAndBuildNumber</key><true/>
</dict>
</plist>
PLIST
            fi

            # Archive
            xcodebuild -workspace "$WORKSPACE" -scheme "$SCHEME" -configuration "$CONFIG" \
              -archivePath "$ARCHIVE_PATH" \
              clean archive | tee xcodebuild-archive.log | grep -E "error:|warning:" -n || true

            # Export IPA
            xcodebuild -exportArchive \
              -archivePath "$ARCHIVE_PATH" \
              -exportOptionsPlist "$EXPORT_PLIST" \
              -exportPath "$EXPORT_DIR" | tee xcodebuild-export.log | grep -E "error:|warning:" -n || true
          '''

					withCredentials([
						string(credentialsId: 'ASC_USER',     variable: 'ASC_USER'),
						string(credentialsId: 'ASC_PASSWORD', variable: 'ASC_PASSWORD')
					]) {
						sh '''
              set -euo pipefail
              IPA_PATH=$(ls build/*.ipa | head -n 1 || true)
              if [ -z "$IPA_PATH" ]; then
                echo "No IPA found in build directory"; exit 1
              fi
              xcrun iTMSTransporter -m upload -assetFile "$IPA_PATH" \
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
			archiveArtifacts artifacts: 'android/app/build/outputs/bundle/release/*.aab', fingerprint: true
		}
		always {
			junit allowEmptyResults: true, testResults: 'android/**/build/test-results/**/*.xml'
		}
	}
}
