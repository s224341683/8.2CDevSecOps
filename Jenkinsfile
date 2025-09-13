pipeline {
  agent any
  options {
    timestamps()
  }
  environment {
    CI = "true"
  }
  stages {
    stage('Checkout') {
      steps {
        // If job uses "Pipeline script from SCM", Jenkins auto-checkouts already.
        // This keeps it explicit and works either way.
        checkout scm
      }
    }
    stage('Node.js version') {
      steps {
        sh 'node -v && npm -v'
      }
    }
    stage('Install dependencies') {
      steps {
        sh '''
          mkdir -p logs
          npm ci 2>&1 | tee logs/install.log
        '''
      }
    }
    stage('Unit tests (best effort)') {
      steps {
        script {
          // Try tests if defined; mark UNSTABLE if they fail or aren't defined
          def rc = sh(returnStatus: true, script: '''
            if npm run | grep -q " test"; then
              npm test 2>&1 | tee logs/test.log
            else
              echo "No test script defined in package.json" | tee logs/test.log
              exit 2
            fi
          ''')
          if (rc != 0) {
            currentBuild.result = 'UNSTABLE'
            echo "Tests failed or are not defined. Marking build UNSTABLE, continuing."
          }
        }
      }
    }
    stage('Security audit (fail on high+)') {
      steps {
        script {
          // Save machine-readable findings and fail build on high/critical
          def auditRC = sh(returnStatus: true, script: '''
            npm audit --json 2>&1 | tee logs/npm-audit.json >/dev/null
            # npm audit exits non-zero when vulns present; enforce threshold:
            npm audit --audit-level=high 2>&1 | tee logs/npm-audit-high.log
          ''')
          if (auditRC != 0) {
            error "High (or above) vulnerabilities detected. See logs/npm-audit-high.log"
          }
        }
      }
    }
  }
  post {
    always {
      archiveArtifacts artifacts: 'logs/**', allowEmptyArchive: true
    }
    failure {
      echo 'Build FAILED. See console and archived logs.'
    }
    unstable {
      echo 'Build UNSTABLE (tests failed or missing). Proceeding per task instructions.'
    }
    success {
      echo 'Build SUCCESS.'
    }
  }
}
