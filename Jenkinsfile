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
				RUBY_VERSION = '3.3.4' // used if rbenv is present
			}
			steps {
				dir('ios') {
					sh '''
            #!/bin/bash
            set -euo pipefail
            export LANG=en_US.UTF-8
            export LC_ALL=en_US.UTF-8

            # Prefer rbenv if present, otherwise use system Ruby
            if command -v rbenv >/dev/null 2>&1; then
              echo "rbenv found: $(command -v rbenv)"
              eval "$(rbenv init - bash)" || eval "$(rbenv init -)" || true
              rbenv install -s "${RUBY_VERSION}"
              rbenv shell   "${RUBY_VERSION}"
            else
              echo "rbenv not found — using system Ruby."
            fi

            echo "Ruby: $(ruby -v || echo 'missing ruby')"
            echo "Gem:  $(gem -v   || echo 'missing rubygems')"

            # Install to user gem dir (no sudo needed)
            export GEM_HOME="$HOME/.gem"
            export PATH="$GEM_HOME/bin:$PATH"

            # Choose a Bundler version compatible with the current Ruby
            # Ruby < 3.0 → use Bundler 2.4.x; Ruby >= 3.0 → use Bundler 2.5.x (or Gemfile.lock value)
            RUBY_MAJOR=$(ruby -e 'v=RUBY_VERSION.split(".").map(&:to_i); puts v[0]' 2>/dev/null || echo 0)
            RUBY_MINOR=$(ruby -e 'v=RUBY_VERSION.split(".").map(&:to_i); puts v[1]' 2>/dev/null || echo 0)

            # Try to read "BUNDLED WITH" from Gemfile.lock
            LOCK_BUNDLER=$(ruby -e 'f="Gemfile.lock"; puts(/BUNDLED WITH\\n\\s+([0-9.]+)/.match(File.read(f))[1] rescue "")' 2>/dev/null || true)

            if [ "$RUBY_MAJOR" -lt 3 ]; then
              # Force a Bundler compatible with Ruby 2.6/2.7
              BUNDLER_VER="${LOCK_BUNDLER:-2.4.22}"
              echo "Using Bundler $BUNDLER_VER for Ruby ${RUBY_MAJOR}.${RUBY_MINOR}"
            else
              BUNDLER_VER="${LOCK_BUNDLER:-2.5.11}"
              echo "Using Bundler $BUNDLER_VER for Ruby ${RUBY_MAJOR}.${RUBY_MINOR}"
            fi

            gem list -i bundler -v "$BUNDLER_VER" >/dev/null 2>&1 || gem install bundler:"$BUNDLER_VER" --no-document

            # Try bundle install; if it fails (e.g. due to lockfile version), fall back to direct Cocoapods install
            set +e
            bundle _${BUNDLER_VER}_ config set --local path 'vendor/bundle'
            bundle _${BUNDLER_VER}_ install --jobs=4
            BUNDLE_RC=$?
            set -e

            if [ $BUNDLE_RC -ne 0 ]; then
              echo "bundle install failed; falling back to installing cocoapods directly"
              gem list -i cocoapods >/dev/null 2>&1 || gem install cocoapods --no-document
              pod install --repo-update
            else
              echo "bundle install succeeded"
              if grep -qi cocoapods Gemfile 2>/dev/null; then
                bundle _${BUNDLER_VER}_ exec pod install --repo-update
              else
                gem list -i cocoapods >/dev/null 2>&1 || gem install cocoapods --no-document
                pod install --repo-update
              fi
            fi

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
              -archivePath "$ARCHIVE_PATH" clean archive \
              | tee xcodebuild-archive.log | grep -E "error:|warning:" -n || true

            # Export
            xcodebuild -exportArchive \
              -archivePath "$ARCHIVE_PATH" \
              -exportOptionsPlist "$EXPORT_PLIST" \
              -exportPath "$EXPORT_DIR" \
              | tee xcodebuild-export.log | grep -E "error:|warning:" -n || true
          '''

					withCredentials([
						string(credentialsId: 'ASC_USER',     variable: 'ASC_USER'),
						string(credentialsId: 'ASC_PASSWORD', variable: 'ASC_PASSWORD')
					]) {
						sh '''
              #!/bin/bash
              set -euo pipefail
              IPA_PATH=$(ls build/*.ipa 2>/dev/null | head -n 1 || true)
              if [ -z "$IPA_PATH" ]; then
                echo "No IPA found in build directory"
                exit 1
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
