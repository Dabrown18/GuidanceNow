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
			steps {
				withEnv([
					// Ensure Node is on PATH for Podfile `node` call; keep gems user-scoped
					"PATH=${env.HOME}/.gem/bin:${tool(name: 'node20', type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation')}/bin:/usr/bin:/bin:/usr/sbin:/sbin",
					"GEM_HOME=${env.HOME}/.gem",
					"GEM_PATH=${env.HOME}/.gem",
					"LANG=en_US.UTF-8",
					"LC_ALL=en_US.UTF-8"
				]) {
					dir('ios') {
						sh '''
              set -euo pipefail

              # Guard: if Xcode isn't present, skip politely
              if ! command -v xcodebuild >/dev/null 2>&1; then
                echo "xcodebuild not found on this agent — skipping iOS stage."
                exit 0
              fi

              echo "Xcode:"
              which xcodebuild
              xcodebuild -version || true

              echo "Ruby & Gem:"
              ruby -v || true
              gem -v  || true

              echo "Node in PATH for CocoaPods/Podfile: $(which node || true)"
              node -v || true

              # --- Bundler in user space (no /Library writes) ---
              BUNDLER_VER=2.4.22
              if ! gem list -i bundler -v "${BUNDLER_VER}" >/dev/null 2>&1; then
                gem install --user-install bundler:"${BUNDLER_VER}" --no-document
                hash -r || true
              fi

              # Install gems into project-local vendor/bundle
              bundle _${BUNDLER_VER}_ config set --local path 'vendor/bundle'
              bundle _${BUNDLER_VER}_ install --jobs=4

              # CocoaPods (from Gemfile if present)
              bundle _${BUNDLER_VER}_ exec pod install --repo-update

              # --- Build settings ---
              WORKSPACE="GuidanceNow.xcworkspace"
              SCHEME="GuidanceNow"
              CONFIG="Release"
              ARCHIVE_PATH="$(pwd)/build/GuidanceNow.xcarchive"
              EXPORT_DIR="$(pwd)/build"
              EXPORT_PLIST="$(pwd)/exportOptions.plist"

              # Create exportOptions if missing (defaults for App Store build)
              if [ ! -f "$EXPORT_PLIST" ]; then
                cat > "$EXPORT_PLIST" <<'PLIST'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key><string>app-store</string>
  <key>compileBitcode</key><false/>
  <key>stripSwiftSymbols</key><true/>
  <key>signingStyle</key><string>automatic</string>
  <key>destination</key><string>export</string>
  <key>uploadSymbols</key><true/>
</dict>
</plist>
PLIST
              fi

              # --- Archive ---
              xcodebuild -workspace "$WORKSPACE" -scheme "$SCHEME" -configuration "$CONFIG" \
                -archivePath "$ARCHIVE_PATH" \
                -allowProvisioningUpdates \
                clean archive | tee xcodebuild-archive.log | grep -E "error:|warning:" -n || true

              # Verify archive exists
              [ -d "$ARCHIVE_PATH" ] || { echo "Archive not created"; exit 1; }

              # --- Export IPA ---
              xcodebuild -exportArchive \
                -archivePath "$ARCHIVE_PATH" \
                -exportOptionsPlist "$EXPORT_PLIST" \
                -exportPath "$EXPORT_DIR" | tee xcodebuild-export.log | grep -E "error:|warning:" -n || true

              IPA_PATH=$(ls "$EXPORT_DIR"/*.ipa | head -n 1 || true)
              if [ -z "$IPA_PATH" ]; then
                echo "No IPA found in $EXPORT_DIR"; exit 1
              fi
              echo "Exported IPA: $IPA_PATH"

              # --- Optional App Store upload ---
              if [ -n "${ASC_USER:-}" ] && [ -n "${ASC_PASSWORD:-}" ] && [ -n "${ITC_PROVIDER:-}" ]; then
                echo "Uploading via iTMSTransporter with ITC_PROVIDER=$ITC_PROVIDER"
                xcrun iTMSTransporter -m upload -assetFile "$IPA_PATH" \
                  -u "$ASC_USER" -p "$ASC_PASSWORD" -itc_provider "$ITC_PROVIDER" -verbose
              else
                echo "Skipping App Store upload (ASC_USER/ASC_PASSWORD/ITC_PROVIDER not set)."
              fi
            '''
					}

					// Show where the IPA landed
					sh 'ls -la ios/build || true'
				}
			}
			// No credentials() bindings here — avoids hard failure if they aren't configured
		}
	}

	post {
		success {
			archiveArtifacts artifacts: 'android/app/build/outputs/bundle/release/*.aab', fingerprint: true
			archiveArtifacts artifacts: 'ios/build/**/*.ipa', fingerprint: true
		}
		always {
			junit allowEmptyResults: true, testResults: 'android/**/build/test-results/**/*.xml'
		}
	}
}
