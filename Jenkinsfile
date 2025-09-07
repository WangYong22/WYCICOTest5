pipeline {
  agent any

  environment {
    KCFG_ID     = 'k8s-jenkins'                 // kubeconfig 凭据ID
    POD_TMPL    = 'macos-pod.yaml'              // Pod 模板
    BUILD_NODE  = "macos-${env.BUILD_TAG}"      // 本次构建唯一节点名
    JENKINS_URL = 'http://17.87.2.137:8080'     // Jenkins 根URL（不要以 / 结尾）
  }

  options { timestamps() }

  stages {

    stage('Create Jenkins Node') {
      steps {
        withCredentials([usernameColonPassword(credentialsId: 'jenkins-api', variable: 'JAUTH')]) {
          script {
            def NODE = env.BUILD_NODE
            def JURL = (env.JENKINS_URL?.trim()) ?: 'http://17.87.2.137:8080'

            // 1) 写入节点 XML（注意 \$ 的转义）
            writeFile file: 'node.xml', text: """
<slave>
  <name>${NODE}</name>
  <description>ephemeral macOS</description>
  <remoteFS>/Users/test/jenkins-agent</remoteFS>
  <numExecutors>1</numExecutors>
  <mode>NORMAL</mode>
  <retentionStrategy class="hudson.slaves.RetentionStrategy\\\$Always"/>
  <launcher class="hudson.slaves.JNLPLauncher">
    <webSocket>true</webSocket>
  </launcher>
  <label>${NODE}</label>
  <nodeProperties/>
</slave>
""".stripIndent()

            // 2) 取 crumb -> /scriptText 创建/更新 -> 覆盖 config.xml -> 解析 JNLP secret（不跑 kubectl）
            sh """#!/bin/bash
set -xeuo pipefail
NODE='${NODE}'
JURL='${JURL}'

# 取 CSRF crumb
if command -v jq >/dev/null 2>&1; then
  CRUMB=\$(curl -fsS -u "\$JAUTH" "\$JURL/crumbIssuer/api/json" | jq -r .crumb)
else
  CRUMB=\$(curl -fsS -u "\$JAUTH" "\$JURL/crumbIssuer/api/json" | sed -n 's/.*"crumb"[[:space:]]*:[[:space:]]*"\\([^"]*\\)".*/\\1/p')
fi
[ -n "\$CRUMB" ] || { echo "Failed to get CRUMB"; exit 1; }

# 写到临时文件，避免 read -d '' 的坑
cat > .scriptText.groovy <<GROOVY
import jenkins.model.Jenkins
import hudson.model.Node
import hudson.slaves.DumbSlave
import hudson.slaves.RetentionStrategy
import hudson.slaves.JNLPLauncher
import java.util.LinkedList

def name   = "${NODE}"            // ← 直接嵌入 bash 里的 NODE 值
def labels = name
def home   = "/Users/test/jenkins-agent"

def j = Jenkins.get()
def n = j.getNode(name)
def launcher = new JNLPLauncher()
launcher.setWebSocket(true)

if (n == null) {
  n = new DumbSlave(name, "ephemeral macOS", home, "1",
    Node.Mode.NORMAL, labels, launcher,
    new RetentionStrategy.Always(), new LinkedList())
  j.addNode(n)
} else {
  n.setLauncher(launcher)
  n.setLabelString(labels)
  n.setNumExecutors(1)
  n.setMode(Node.Mode.NORMAL)
  j.save()
}
println "READY"
GROOVY

export NODE_NAME="\$NODE"

# 执行 /scriptText
RESP=\$(curl -fsS -u "\$JAUTH" -H "Jenkins-Crumb: \$CRUMB" \\
  --data-urlencode script@.scriptText.groovy \\
  "\$JURL/scriptText")
echo "\$RESP" | grep -q READY || { echo "scriptText create node failed"; echo "\$RESP" | head -200; exit 1; }

# 覆盖节点 config.xml（幂等）
curl -fsS -u "\$JAUTH" -H "Jenkins-Crumb: \$CRUMB" \\
     -X POST -H 'Content-Type: application/xml' \\
     --data-binary @node.xml \\
     "\$JURL/computer/\$NODE/config.xml"

# 解析 JNLP secret（新旧路径兼容）
sleep 2
HTTP1=\$(curl -u "\$JAUTH" -w '%{http_code}' -fsSLo .jnlp1 "\$JURL/computer/\$NODE/jenkins-agent.jnlp" || true)
HTTP2=\$(curl -u "\$JAUTH" -w '%{http_code}' -fsSLo .jnlp2 "\$JURL/computer/\$NODE/slave-agent.jnlp"   || true)
echo "jenkins-agent.jnlp http=\$HTTP1, slave-agent.jnlp http=\$HTTP2"

SECRET=""
SECRET=\$(grep -hoE -- '--secret [0-9a-f]{64,}' .jnlp1 .jnlp2 2>/dev/null | head -1 | awk '{print \$2}') || true
if [ -z "\$SECRET" ]; then
  SECRET=\$(grep -hoE -- '<argument>[0-9a-f]{64,}</argument>' .jnlp1 .jnlp2 2>/dev/null | head -1 | sed -E 's#</?argument>##g') || true
fi
[ -n "\$SECRET" ] || { echo "no JNLP secret parsed"; head -50 .jnlp1 .jnlp2 2>/dev/null || true; exit 1; }
echo "\$SECRET" > .jnlp.secret
echo "Parsed JNLP secret length=\${#SECRET}"
"""

// 关键：从文件读回到 Groovy 变量
def SECRET = readFile('.jnlp.secret').trim()
echo "Groovy got SECRET length=${SECRET.length()}"


// 3) 现在进入 withKubeConfig，下发 K8s Secret 时，使用 Groovy 变量插值
withKubeConfig([credentialsId: env.KCFG_ID]) {
  sh """#!/bin/bash
set -xeuo pipefail
kubectl -n ci delete secret jenkins-macos-${NODE} --ignore-not-found || true
kubectl -n ci create secret generic jenkins-macos-${NODE} \\
  --from-literal=JENKINS_URL=${JURL} \
  --from-literal=JENKINS_NODE_NAME=${NODE} \
  --from-literal=JENKINS_SECRET=${SECRET}  \
  --from-literal=VM_USER=test \
  --from-literal=VM_PASS=test 
"""
            }
          }
        }
      }
    }

    stage('Launch VM Pod') {
      steps {
        sh 'cp /var/jenkins_home/pod-templates/macos-pod.yaml .'
        withKubeConfig([credentialsId: env.KCFG_ID]) {
          sh """
            sed -e "s/__BUILD_ID__/${BUILD_NODE}/g" \
                -e "s/__HOST_PORT__/30222/g"  \
                -e "s/__VZ_USER__/test/g" \
                -e "s/__VZ_PASS__/test/g" macos-pod.yaml > rendered.yaml

            echo '===== rendered.yaml (compute env) ====='
            awk '/name: compute/{f=1} f{print} /name: jenkins-bootstrap/{f=0}' rendered.yaml
            echo '======================================'

            kubectl -n ci apply -f rendered.yaml
          """
          
          sh '''
            kubectl -n ci wait --for=condition=Initialized --timeout=120s pod/macos-build-${BUILD_NODE} || true

            kubectl -n ci describe pod macos-build-${BUILD_NODE} | sed -n '/Events:/,$p'
            kubectl -n ci get pod macos-build-${BUILD_NODE} -o wide || true

            kubectl -n ci get pod macos-build-${BUILD_NODE} -o json \
              | jq '.spec.containers[] | select(.name=="jenkins-bootstrap") | {envFrom: .envFrom, env: .env}' || true

            kubectl -n ci get secret jenkins-macos-${BUILD_NODE} -o jsonpath='{.data.JENKINS_URL}' | base64 -d; echo
            kubectl -n ci get secret jenkins-macos-${BUILD_NODE} -o jsonpath='{.data.JENKINS_SECRET}' | base64 -d | awk '{print "LEN=" length}'

        
          '''
          sh '''
            POD=macos-build-${BUILD_NODE}

            # 每个容器的等待原因/信息
            kubectl -n ci get pod "$POD" -o jsonpath='{range .status.containerStatuses[*]}{.name}{" => "}{.state.waiting.reason}{" : "}{.state.waiting.message}{"\n"}{end}' || true
            echo

            # 只看 jenkins-bootstrap 的详细等待对象
            kubectl -n ci get pod "$POD" -o json \
              | jq '.status.containerStatuses[] | select(.name=="jenkins-bootstrap")' || true

            # 相关事件（按时间排序）
            kubectl -n ci get events --field-selector involvedObject.name="$POD" --sort-by=.lastTimestamp || true
          '''
          sh '''
            kubectl -n ci get pod macos-build-${BUILD_NODE} -o yaml \
              | sed -n "/containerStatuses:/,/^$/p" || true
          '''
          sh """
            kubectl -n ci get pod macos-build-${BUILD_NODE} -o wide || true
          """
        }
      }
    }
  
	stage('Cred debug: ssh private key (137ssh)') {
	  steps {
		withCredentials([sshUserPrivateKey(credentialsId: '137ssh',
										   keyFileVariable: 'SSH_KEY',
										   usernameVariable: 'SSH_USER')]) {
		  sh '''#!/bin/bash
			set -euo pipefail
			echo "[DEBUG] local host: $(hostname)"
			
			ssh -i "$SSH_KEY" -o IdentitiesOnly=yes -o StrictHostKeyChecking=no \
			   "$SSH_USER@17.87.2.137" 'echo "[REMOTE] ok from $(hostname)"'
			'''
			}
	  }
	}


	 stage('Build on macOS') {
	  agent { label "${env.BUILD_NODE}" }
	
	  steps {
	    withCredentials([string(credentialsId: 'macos-login-keychain-pass', variable: 'KEYCHAIN_PASS')]) {
		  sh '''#!/bin/bash
		  set -euo pipefail
		  set +u
		  LOGIN_KC="${HOME}/Library/Keychains/login.keychain-db"
		  KEYCHAIN_PASS="${KEYCHAIN_PASS:-}"
		  set -u
		
		  : "${KEYCHAIN_PASS:?KEYCHAIN_PASS is empty — configure Jenkins credential 'macos-login-keychain-pass'}"
		
		  [ -f "$LOGIN_KC" ] || security create-keychain -p "${KEYCHAIN_PASS}" "$LOGIN_KC"
		  security unlock-keychain -p "${KEYCHAIN_PASS}" "$LOGIN_KC"
		  security default-keychain -s "$LOGIN_KC"
		  security list-keychains -d user -s "$LOGIN_KC"
		  security set-key-partition-list -S apple-tool:,apple:,codesign: -s -k "${KEYCHAIN_PASS}" "$LOGIN_KC"
		  '''
		}
	  
		// 1) 连通性
		sh '''#!/bin/bash
		set -euo pipefail
		hostname
		sw_vers
		'''
	
		// 2) 拉取 provisioning profile（用 Jenkins 中 ID=137ssh 的私钥）
		withCredentials([sshUserPrivateKey(credentialsId: '137ssh',
										   keyFileVariable: 'SSH_KEY',
										   usernameVariable: 'SSH_USER')]) {
		  sh '''#!/bin/bash
			set -euo pipefail
			REMOTE="17.87.2.137"
			REMOTE_FILE="/Users/mdsadmin/WangYongiOS1.mobileprovision"     # <— 确认远端真实文件名
			LOCAL="/tmp/WangYongiOS1.mobileprovision"
			
			echo "[SCP] $REMOTE:$REMOTE_FILE -> $LOCAL"
			scp -i "$SSH_KEY" -o IdentitiesOnly=yes -o StrictHostKeyChecking=no \
				"$SSH_USER@$REMOTE:$REMOTE_FILE" "$LOCAL"
			'''
		  }
	
		// 3) 安装 profile（解析 UUID）
		sh '''#!/bin/bash
			set -euo pipefail
			PROFILE="/tmp/WangYongiOS1.mobileprovision"
			PLIST="/tmp/WangYongiOS1.plist"
			SECERR="/tmp/security.err"
			
			echo "==[PARSE] security cms -> $PLIST =="
			# 先尝试按 CMS 解包；失败时把错误打出来，并在是 XML 的情况下兜底
			if /usr/bin/security cms -D -i "$PROFILE" > "$PLIST" 2>"$SECERR"; then
			  echo "security cms OK"
			else
			  echo "security cms failed:"
			  cat "$SECERR" || true
			  if file "$PROFILE" | grep -qi 'XML'; then
				echo "Detected XML plist; using it directly"
				cp "$PROFILE" "$PLIST"
			  else
				exit 1
			  fi
			fi
			
			echo "==[VERIFY] plutil -lint =="
			/usr/bin/plutil -lint "$PLIST"
	
			echo "==[UUID] PlistBuddy =="
			UUID=$(/usr/libexec/PlistBuddy -c 'Print :UUID' "$PLIST")
			echo "[PROFILE] UUID=$UUID"
			
			echo "==[INSTALL]=="
			/bin/mkdir -p "$HOME/Library/MobileDevice/Provisioning Profiles"
			/usr/bin/install -m 0644 "$PROFILE" "$HOME/Library/MobileDevice/Provisioning Profiles/${UUID}.mobileprovision"
			echo "Installed: $HOME/Library/MobileDevice/Provisioning Profiles/${UUID}.mobileprovision"
				
		# 4) 构建（保持纯 shell；不要塞 Groovy 语句）
				
		CPROVISIONING_PROFILE_NAME="WangYongiOS1"
		Project_Name="WYCICOTest5"
		BundleID="com.WangYong2.WYCICDiOSTest"                         # <— 与后台保持一致（小写 w）
		DEVELOPMENT_TEAM="64KDUQCYEB"
		CODE_SIGN_IDENTITY="Apple Development: Yong Wang (LFBT5QQQ6J)"
		Configuration="Debug"
		project_scheme="WYCICOTest5"	
		project_workspace="${Project_Name}.xcworkspace"	
		project_proj="${Project_Name}.xcodeproj"	
		build_dir="${WORKSPACE}/build"
		archive_path="${build_dir}/${project_scheme}.xcarchive"
		export_path="${HOME}/project/build/${Project_Name}"
		exportOptionsPlist="${build_dir}/${project_scheme}.plist"	
		
		mkdir -p "${build_dir}" "${export_path}"
				
		# 记录变量
		cat > BuildVariable <<EOF
		project_workspace=${project_workspace}
		project_scheme=${project_scheme}
		export_path=${export_path}
		build_dir=${build_dir}
		archive_path=${archive_path}
		exportOptionsPlist=${exportOptionsPlist}
EOF
		
		if [ -f "$project_workspace" ]; then
		  echo "[BUILD] Use workspace: $project_workspace"
		  xcodebuild -list -workspace "$project_workspace" || true
		  SRC_OPTS=(-workspace "$project_workspace")
		else
		  echo "[BUILD] Use project: $project_proj"
		  xcodebuild -list -project "$project_proj" || true
		  SRC_OPTS=(-project "$project_proj")
		fi
		
		echo "check3"
		
		# 生成 exportOptionsPlist（手工签名，用 Name 绑定 profile）
		cat > "${exportOptionsPlist}" <<PL
		<?xml version="1.0" encoding="UTF-8"?>
		<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
		<plist version="1.0"><dict>
		  <key>signingStyle</key><string>manual</string>
		  <key>method</key><string>development</string>
		  <key>signingCertificate</key><string>Apple Development</string>
		  <key>provisioningProfiles</key><dict>
			<key>${BundleID}</key><string>${CPROVISIONING_PROFILE_NAME}</string>
		  </dict>
		  <key>stripSwiftSymbols</key><true/>
		  <key>compileBitcode</key><true/>
		</dict></plist>
PL
		
		
		
		
		
		echo " 构建 Archive "
		xcodebuild \
		  "${SRC_OPTS[@]}" \
		  -scheme "$project_scheme" \
		  -configuration "$Configuration" \
		  clean archive \
		  -archivePath "$archive_path" \
		  -destination 'generic/platform=iOS' \
		  DEVELOPMENT_TEAM="$DEVELOPMENT_TEAM" \
		  CODE_SIGN_IDENTITY="$CODE_SIGN_IDENTITY" \
		  PRODUCT_BUNDLE_IDENTIFIER="$BundleID" \
		  PROVISIONING_PROFILE_SPECIFIER="$CPROVISIONING_PROFILE_NAME" \
		  -allowProvisioningUpdates \
		  -allowProvisioningDeviceRegistration \
		  -UseModernBuildSystem=NO \
		  -quiet
		  
		echo " 导出 IPA"		
		xcodebuild -exportArchive \
		  -archivePath "$archive_path" \
		  -exportOptionsPlist "$exportOptionsPlist" \
		  -exportPath "$export_path" \
		  -quiet
		
		echo "[BUILD] exported:"
		ls -lh "$export_path" || true
		'''
		  }
		
		 //  5) 在这个 stage 的 post 里收集产物（仍在 macOS 节点上）"	
		  post {
			always {
			  script {
			  withCredentials([sshUserPrivateKey(credentialsId: '137ssh',
                                   keyFileVariable: 'SSH_KEY',
                                   usernameVariable: 'SSH_USER')]) {
		sh '''#!/bin/bash
		set -euo pipefail
		
		REMOTE="17.87.2.137"
		REMOTE_BASE="/Users/mdsadmin/Documents"
		RUN_DIR="${JOB_BASE_NAME}#${BUILD_NUMBER}"
		REMOTE_DIR="${REMOTE_BASE}/${RUN_DIR}"
		
		echo "读取构建阶段写下的变量；做兜底，避免 set -u 触发"		
		[ -f BuildVariable ] && source BuildVariable || true
		export_path=${export_path:-}
		archive_path=${archive_path:-}
		project_scheme=${project_scheme:-app}
		
		echo "[REMOTE] mkdir -p ${REMOTE_DIR}"
		ssh -i "$SSH_KEY" -o IdentitiesOnly=yes -o StrictHostKeyChecking=no \
			"$SSH_USER@$REMOTE" "mkdir -p \"${REMOTE_DIR}\""
		
		echo "传 ipa"
		if [ -n "$export_path" ] && ls "$export_path"/*.ipa >/dev/null 2>&1; then
		  echo "[UPLOAD] ipa -> ${REMOTE}:${REMOTE_DIR}"
		  scp -i "$SSH_KEY" -o IdentitiesOnly=yes -o StrictHostKeyChecking=no \
			  "$export_path"/*.ipa \
			  "$SSH_USER@$REMOTE:${REMOTE_DIR}/"
		else
		  echo "[UPLOAD] no ipa found under ${export_path}"
		fi
		
		echo " .xcarchive（用 rsync 更快且可断点续传） "
		if [ -n "$archive_path" ] && [ -d "$archive_path" ]; then
		  echo "[UPLOAD] xcarchive -> ${REMOTE}:${REMOTE_DIR}/${project_scheme}.xcarchive/"
		  rsync -a \
			-e "ssh -i $SSH_KEY -o IdentitiesOnly=yes -o StrictHostKeyChecking=no" \
			"$archive_path"/ \
			"$SSH_USER@$REMOTE:${REMOTE_DIR}/${project_scheme}.xcarchive/"
		else
		  echo "[UPLOAD] no xcarchive dir at ${archive_path}"
		fi
		
		echo "附带上传变量文件，便于远端排查"
		[ -f BuildVariable ] && scp -i "$SSH_KEY" -o IdentitiesOnly=yes -o StrictHostKeyChecking=no \
			BuildVariable "$SSH_USER@$REMOTE:${REMOTE_DIR}/" || true
		
		echo "[DONE] artifacts uploaded to ${REMOTE}:${REMOTE_DIR}"
		'''
		}
			  }
			}
		  }
		}
      }
  
  
  
  post {
    always {   
    	 withKubeConfig([credentialsId: env.KCFG_ID]) {
    	   sh "kubectl -n ci delete pod macos-build-${BUILD_NODE} --ignore-not-found"
       	   sh "kubectl -n ci delete secret jenkins-macos-${BUILD_NODE} --ignore-not-found"
      }
    }
  }
}