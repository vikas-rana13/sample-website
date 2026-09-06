pipeline {
    agent any

    // 1. Interactive UI parameters (Build/Test removed)
    parameters {
        booleanParam(name: 'DEPLOY', defaultValue: false, description: 'Provision AWS stack and deploy HTML website?')
        string(name: 'APP_NAME', defaultValue: 'simple-html-site', description: 'Unique name for your website and AWS stack')
    }

    environment {
        AWS_REGION = 'ap-southeast-2'
        // Dynamically injects Apple Silicon and Intel macOS brew paths into Jenkins' PATH
        PATH = "/opt/homebrew/bin:/usr/local/bin:${env.PATH}"
    }

    stages {
        // 2. Provision AWS Infrastructure using CloudFormation
        stage('AWS CloudFormation Provisioning') {
            when { expression { params.DEPLOY } }
            steps {
                echo "=== Deploying/Updating AWS Infrastructure Stack: ${params.APP_NAME}-stack ==="
                sh """
                aws cloudformation deploy \
                  --stack-name "${params.APP_NAME}-stack" \
                  --template-file cloudformation/ec2-provision.yaml \
                  --parameter-overrides AppName="${params.APP_NAME}" KeyName="ec2-key" \
                  --capabilities CAPABILITY_IAM \
                  --region ${env.AWS_REGION}
                """
            }
        }

        // 3. Securely deploy index.html using the SSH Credentials Store
        stage('Deploy Website via SCP') {
            when { expression { params.DEPLOY } }
            steps {
                echo "=== Fetching Public IP for Stack: ${params.APP_NAME}-stack ==="
                script {
                    // Extract the EC2 instance's dynamic public IP
                    def ec2Ip = sh(
                        script: "aws cloudformation describe-stacks --stack-name \"${params.APP_NAME}-stack\" --query \"Stacks.Outputs[?OutputKey=='EC2PublicIP'].OutputValue\" --output text --region ${env.AWS_REGION}",
                        returnStdout: true
                    ).trim()
                    
                    echo "Target EC2 Public IP: ${ec2Ip}"
                    
                    echo "=== Copying index.html to EC2 Server ==="
                    // Securely wraps the SCP command using your saved 'ec2-ssh-key' credentials
                    sshagent(credentials: ['ec2-ssh-key']) {
                        sh "scp -o StrictHostKeyChecking=no index.html ec2-user@${ec2Ip}:/var/www/html/index.html"
                    }
                }
                echo "=== Deployment Completed Successfully! ==="
            }
        }
    }
}
