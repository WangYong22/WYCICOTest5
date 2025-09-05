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

    stage('Build on macOS') {
      agent { label "${env.BUILD_NODE}" }    // 等 inbound agent 上线
      steps {
        sh '''#!/bin/bash
    set -euo pipefail
    xcodebuild -version || true
    # TODO: 放实际构建命令
          
    #脚本开始————————————————
			
			
			hostname
			sw_vers
			
			# Profile 文件名字 
			CPROVISIONING_PROFILE_NAME="WYCICOTest5V2"
			
			#工程名字(Target名字)
			Project_Name="WYCICOTest5"
			
			# AdHoc版本的Bundle ID
			BundleID=com.wangyong2.WYCICOTest5
			
			#DEVELOPMENT_TEAM编码
			DEVELOPMENT_TEAM="64KDUQCYEB"
			
			# Profile 文件 UUID
			PROFILE_UUID="a838c815-d2ed-4c15-aa6c-ea1fedcbc049"
			
			# 代码签名标识
			CODE_SIGN_IDENTITY="Apple Development:Yong Wang(LFBT5QQQ6J)"
			
			
			# 打包的环境，正式环境为空，可选 {'Develop';  'Release';  ''}
			APP_ENV="Develop"
			
			#配置环境，Release或者Debug
			Configuration=Debug
			
			# 项目Scheme名称，通常和工程名字相同
			project_scheme="WYCICOTest5"
			
			# 打开本地钥匙串的密码，电脑管理员账户的密码
			KCpassword=test
			
			# build 路径
			project_workspace="${Project_Name}.xcworkspace"
			
			
			export_path="${HOME}/project/build/${Project_Name}"
			export_name="${export_path}/${project_scheme}_${APP_VERSION}.ipa"
			
			# archive 目标路径
			build_dir="${WORKSPACE}/build"
			archive_path="${build_dir}/${project_scheme}.xcarchive"
			
			#plist文件
			exportOptionsPlist="${build_dir}/${project_scheme}.plist"
			
			
			# 构建变量写入文件
			echo "APP_VERSION=${APP_VERSION}"
			project_workspace=${project_workspace}
			project_scheme=${project_scheme}
			export_path=${export_path}
			export_name=${export_name}
			build_dir=${build_dir}
			archive_path=${archive_path}
			exportOptionsPlist=${exportOptionsPlist} > BuildVariable
			
			
			
			func_export_plist(){
			echo '<?xml version="1.0" encoding="UTF-8"?>
			<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
			<plist version="1.0">
			<dict>
					<key>signingStyle</key>
					<string>manual</string>
					<key>method</key>
					<string>development</string>
					<key>signingCertificate</key>
					<string>Mac Developer</string>
					<key>provisioningProfiles</key>
					<dict>
							<key>'${BundleID}'</key>
							<string>'${PROFILE_UUID}'</string>
					</dict>
					<key>iCloudContainerEnvironment</key>
					<string>Development</string>
					<key>stripSwiftSymbols</key>
					<true/>
					<key>compileBitcode</key>
					<true/>
			</dict>
			</plist>' > ${exportOptionsPlist}
			}
			 
			func_build(){
			   
				# login证书解锁
				security unlock-keychain -p "$KCpassword" ${HOME}/Library/Keychains/login.keychain-db    
			   
			   #给codesign 访问 login 证书权限
			   security set-key-partition-list -S apple-tool:,apple:,codesign: -s -k "$KCpassword" ${HOME}/Library/Keychains/login.keychain-db
			   
			   #测试 codesign能否正常使用证书
			   cp "/usr/bin/true" "MyTrue"
			   codesign -s "Apple Development" -f "MyTrue" 
			   
				
				security show-keychain-info ${HOME}/Library/Keychains/login.keychain-db
				
				#查看可用证书
				security find-identity -vp codesigning
			
			  # 清理
			  #  xcodebuild clean -scheme $project_scheme -configuration $Configuration 
			  # 构建 archive 包
			  xcodebuild archive  \
				-scheme $project_scheme \
				-configuration $Configuration \
				clean archive \
				-archivePath $archive_path \
				BUILD_DIR="$build_dir"  \
				DEVELOPMENT_TEAM="$DEVELOPMENT_TEAM" \
				-allowProvisioningUpdates \
				-allowProvisioningDeviceRegistration \
				-UseModernBuildSystem=NO \
				BUILD_DIR="$build_dir"  \
				PROVISIONING_PROFILE_SPECIFIER="${CPROVISIONING_PROFILE_NAME}" \
				PROVISIONING_PROFILE="${PROFILE_UUID}" \
			    PRODUCT_BUNDLE_IDENTIFIER="${BundleID}" \
				-quiet
			  
			  #输出ipa包
			  xcodebuild -exportArchive -archivePath $archive_path -exportOptionsPlist $exportOptionsPlist -exportPath $export_path
			}
			
			cd ${WORKSPACE}
			mkdir -p build
			# 编译
			func_export_plist
			func_build
			
	def TARGET_ROOT = "/var/jenkins_home/buildDir"
    	def RUN_DIR = "${env.JOB_BASE_NAME}#${env.BUILD_NUMBER}"
    	def TARGET_DIR = "${TARGET_ROOT}/${RUN_DIR}"

    	
		
	mkdir -p "${TARGET_DIR}"
	# 读取我们在 sh 里写的变量文件
	source BuildVariable

	# 拷贝 ipa
	if ls "${export_path}"/*.ipa >/dev/null 2>&1; then
	  cp "${export_path}"/*.ipa "${TARGET_DIR}/"
	fi
	# 拷贝 .xcarchive
	if [ -d "${archive_path}" ]; then
	  cp -R "${archive_path}" "${TARGET_DIR}/"
	fi
	  cp BuildVariable "${TARGET_DIR}/" || true
  """

   archiveArtifacts artifacts: "buildDir/${env.JOB_BASE_NAME}#${env.BUILD_NUMBER}/**/*",
		fingerprint: true,
		onlyIfSuccessful: false
			#脚本结束————————————————
           
        '''
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