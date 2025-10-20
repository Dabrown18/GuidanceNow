pipeline {
	agent any

	// Make sure you have a NodeJS tool in "Manage Jenkins → Global Tool Configuration → NodeJS"
	// with the name 'node20-20.19.5' or change the line below to match your tool name.
	tools {
		nodejs 'node20-20.19.5'
	}

	options {
		timestamps()
		ansiColor('xterm')
		buildDiscarder(logRotator(numToKeepStr: '20'))
	}

	environment {
		// Adjust these paths for your Jenkins mac build node if needed
		ANDROID_SDK_ROOT = "${env.HOME}/Library/Android/sdk"
		JAVA_HOME        = "/Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home"

		// iOS vars, update if you use iOS builds here
		IOS_WORKSPACE = "ios/GuidanceNow.xcworkspace"
		IOS_SCHEME   = "GuidanceNow"
		IOS_ARCHIVE  = "ios/build/GuidanceNow.xcarchive"
		IOS_EXPORT   = "ios/build/export"
	}

	stages {
		stage('Checkout') {
			steps {
				// Uses the SCM configured by the Multibranch/SCM job
				checkout scm
				sh 'git rev-parse --short HEAD || true'
			}
		}

		stage('Node & Yarn install') {
			steps {
				// Verify Node tool is actually on PATH
				sh '''
          set -e
          echo "Node version:"
          node -v
          echo "NPM version:"
          npm -v

          echo "Enable Corepack and prepare Yarn"
          corepack enable
          corepack prepare yarn@stable --activate

          echo "Install JS deps"
          yarn --version
          yarn install --frozen-lockfile
        '''
			}
		}

		stage('Android Release Build') {
			steps {
				sh '''
          set -e
          echo "JAVA_HOME: $JAVA_HOME"
          echo "ANDROID_SDK_ROOT: $ANDROID_SDK_ROOT"

          cd android
          chmod +x ./gradlew
          ./gradlew --version

          echo "Cleaning..."
          ./gradlew clean

          echo "Assembling release (AAB)..."
          ./gradlew bundleRelease

          echo "Assembling release (APK, optional)..."
          ./gradlew assembleRelease || true
        '''
			}
			post {
				always {
					script {
						def aab = sh(script: "ls -1 android/app/build/outputs/bundle/release/*.aab 2>/dev/null | head -n1", returnStdout: true).trim()
						def apk = sh(script: "ls -1 android/app/build/outputs/apk/release/*.apk 2>/dev/null | head -n1", returnStdout: true).trim()
						if (aab) {
							archiveArtifacts artifacts: 'android/app/build/outputs/bundle/release/*.aab', fingerprint: true
						}
						if (apk) {
							archiveArtifacts artifacts: 'android/app/build/outputs/apk/release/*.apk', fingerprint: true
						}
					}
				}
			}
		}

		stage('iOS Archive + Upload') {
			when {
				// Only attempt on mac agents that have Xcode, tweak as you like
				expression { isUnix() && sh(script: "uname", returnStdout: true).trim() == 'Darwin' }
			}
			steps {
				// If you use Fastlane, swap these xcodebuild commands for your lanes.
				withEnv(["APP_STORE_CONNECT_API_KEY_JSON=${WORKSPACE}/ios/AuthKey.json"]) {
					sh '''
            set -e
            cd ios

            # If you’re using CocoaPods
            if [ -f "Podfile" ]; then
              echo "Installing CocoaPods"
              which pod || sudo gem install cocoapods || true
              pod install --repo-update
            fi

            echo "Xcode version:"
            xcodebuild -version

            echo "Archiving..."
            xcodebuild \
              -workspace "${IOS_WORKSPACE##ios/}" \
              -scheme "${IOS_SCHEME}" \
              -configuration Release \
              -derivedDataPath build/DerivedData \
              -archivePath "../${IOS_ARCHIVE##ios/}" \
              clean archive

            echo "Exporting IPA..."
            mkdir -p "${IOS_EXPORT}"
            cat > ExportOptions.plist <<'PLIST'
            <?xml version="1.0" encoding="UTF-8"?>
            <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
            <plist version="1.0">
            <dict>
              <key>method</key><string>app-store</string>
              <key>compileBitcode</key><false/>
              <key>destination</key><string>export</string>
              <key>manageAppVersionAndBuildNumber</key><true/>
              <key>signingStyle</key><string>automatic</string>
            </dict>
            </plist>
            PLIST

            xcodebuild -exportArchive \
              -archivePath "../${IOS_ARCHIVE##ios/}" \
              -exportPath "${IOS_EXPORT}" \
              -exportOptionsPlist ExportOptions.plist

            echo "Generated IPA(s):"
            ls -lah "${IOS_EXPORT}" || true
          '''
				}
			}
			post {
				always {
					archiveArtifacts artifacts: 'ios/build/export/*.ipa, ios/build/*.xcarchive', fingerprint: true, onlyIfSuccessful: false
				}
			}
		}
	}

	post {
		success {
			echo 'Build completed successfully.'
		}
		failure {
			echo 'Build failed, please check logs.'
		}
		always {
			// Keep useful logs and lockfiles for debugging
			archiveArtifacts artifacts: 'yarn.lock, package.json, android/**/outputs/**, ios/build/**, **/*.log', fingerprint: true, onlyIfSuccessful: false
		}
	}
}
