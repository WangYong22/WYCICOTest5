pipeline {
  agent any
  // 如果你不想在 “Triggers” 页面勾选，也可用下面这行（两者二选一）
  // triggers { githubPush() }

  options { timestamps() }

  stages {
    stage('Checkout') {
      steps {
        // 若在 Job 里关闭了 “Lightweight checkout”，这里可省略
        checkout scm
      }
    }

    stage('Build') {
      steps {
        sh 'echo Hello from Pipeline'
      }
    }
  }
}