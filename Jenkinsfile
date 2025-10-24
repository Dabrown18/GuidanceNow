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
					// Ensure Node is on PATH for Podfile's `node` calls
					withEnv(["PATH+NODE=${tool name: 'node20', type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation'}/bin"]) {
						sh '''
              set -euo pipefail
              export LANG=en_US.UTF-8
              export LC_ALL=en_US.UTF-8

              echo "Xcode:"
              which xcodebuild
              xcodebuild -version || true

              echo "Node on PATH for CocoaPods:"
              which node || true
              node -v || true
              npm -v || true

              echo "Ruby & Gem:"
              ruby -v
              gem -v

              # Install gems to the user dir and expose their bin/ on PATH
              export GEM_HOME="$HOME/.gem"
              export GEM_PATH="$GEM_HOME"
              export PATH="$GEM_HOME/bin:$PATH"

              # Read Bundler version from Gemfile.lock if present; otherwise use one compatible with Ruby 2.6
              LOCK_BUNDLER="$(ruby -e 'begin; f="Gemfile.lock"; m=/BUNDLED WITH\\n\\s+([0-9.]+)/.match(File.read(f)); puts(m ? m[1] : ""); rescue; puts ""; end')"
              if [ -z "$LOCK_BUNDLER" ]; then
                BUNDLER_VER="2.4.22"   # 2.5+ requires Ruby >= 3.0
              else
                BUNDLER_VER="$LOCK_BUNDLER"
              fi
              echo "Using Bundler ${BUNDLER_VER}"

              # Use Bundler already available (installed in previous runs) via GEM_HOME bin
              hash -r

              # Install gems into project-local vendor/bundle and run Pods via Bundler
              bundle _${BUNDLER_VER}_ config set --local path 'vendor/bundle'
              bundle _${BUNDLER_VER}_ install --jobs=4

              # CocoaPods via Bundler (so we don't rely on PATH to find 'pod')
              bundle _${BUNDLER_VER}_ exec pod install --repo-update

              # Ensure exportOptions.plist exists
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
                clean archive

              # Export IPA
              xcodebuild -exportArchive \
                -archivePath "$ARCHIVE_PATH" \
                -exportOptionsPlist "$EXPORT_PLIST" \
                -exportPath "$EXPORT_DIR"
            '''
					}

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
