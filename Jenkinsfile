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

            echo "Xcode:"
            which xcodebuild
            xcodebuild -version || true

            echo "Ruby & Gem:"
            ruby -v || true
            gem -v || true

            # Read Bundler version from Gemfile.lock, safely on Ruby 2.6
            LOCK_BUNDLER="$(ruby -e 'begin; f=\"Gemfile.lock\"; m=/BUNDLED WITH\\n\\s+([0-9.]+)/.match(File.read(f)); puts(m ? m[1] : \"\"); rescue; puts \"\"; end' || true)"

            RUBY_MAJOR="$(ruby -e 'v=RUBY_VERSION.split(\".\").map(&:to_i); puts v[0]' )"
            RUBY_MINOR="$(ruby -e 'v=RUBY_VERSION.split(\".\").map(&:to_i); puts v[1]' )"

            # Default Bundler for Ruby 2.6; override with lockfile if compatible
            BUNDLER_VER="$LOCK_BUNDLER"
            if [ -z "$BUNDLER_VER" ]; then
              BUNDLER_VER="2.4.22"
            fi

            # If we're on Ruby < 3 and lockfile asks for >= 2.5 (incompatible), force 2.4.22
            if [ "$RUBY_MAJOR" -lt 3 ]; then
              case "$BUNDLER_VER" in
                2.5.*|2.6.*|2.7.*|3.*) BUNDLER_VER="2.4.22" ;;
              esac
            fi

            echo "Using Bundler $BUNDLER_VER (Ruby ${RUBY_MAJOR}.${RUBY_MINOR})"
            gem list -i bundler -v "$BUNDLER_VER" >/dev/null || gem install bundler:"$BUNDLER_VER" --no-document

            # Install gems into vendor/bundle (project-local)
            set +e
            bundle _${BUNDLER_VER}_ config set --local path 'vendor/bundle'
            bundle _${BUNDLER_VER}_ install --jobs=4
            RC=$?
            set -e
            if [ "$RC" -ne 0 ]; then
              echo "bundle install failed with Bundler ${BUNDLER_VER}"
              exit $RC
            fi
            echo "bundle install succeeded"

            # CocoaPods via Bundler (ensures 'pod' is on PATH)
            echo "Running pod install via Bundler…"
            bundle _${BUNDLER_VER}_ exec pod --version
            bundle _${BUNDLER_VER}_ exec pod install --repo-update

            # Ensure exportOptions.plist exists (adjust method as needed)
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

            echo "Archiving…"
            xcodebuild -workspace "$WORKSPACE" -scheme "$SCHEME" -configuration "$CONFIG" \
              -archivePath "$ARCHIVE_PATH" \
              clean archive | tee xcodebuild-archive.log

            echo "Exporting IPA…"
            xcodebuild -exportArchive \
              -archivePath "$ARCHIVE_PATH" \
              -exportOptionsPlist "$EXPORT_PLIST" \
              -exportPath "$EXPORT_DIR" | tee xcodebuild-export.log
          '''

					withCredentials([
						string(credentialsId: 'ASC_USER',     variable: 'ASC_USER'),
						string(credentialsId: 'ASC_PASSWORD', variable: 'ASC_PASSWORD')
					]) {
						sh '''
              set -euo pipefail
              IPA_PATH=$(ls build/*.ipa | head -n 1 || true)
              if [ -z "$IPA_PATH" ]; then
                echo "No IPA found in build directory"
                exit 1
              fi
              echo "Uploading $IPA_PATH to App Store Connect…"
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
