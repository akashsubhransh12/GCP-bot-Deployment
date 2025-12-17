pipeline {
    agent any

    parameters {
        choice(name: 'PRODUCT', choices: ['Agoda', 'Booking'], description: 'Select product to deploy')
        string(name: 'DLL_FILE', defaultValue: '', description: 'Enter DLL file name with extension')
    }

    environment {
        UPLOAD_DIR = "${WORKSPACE}"
        BUCKET = "gs://rankbot-dll-bucket"
    }

    stages {

        stage('Resolve Product Configuration') {
            steps {
                script {
                    def products = [
                        'Agoda':   [enabled: true, ip: '10.10.10.54', dest: 'C:\\RankBot\\agoda',   cred: 'agoda-creds'],
                        'Booking': [enabled: true, ip: '10.10.10.63', dest: 'C:\\RankBot\\booking', cred: 'booking-creds']
                    ]

                    def cfg = products[params.PRODUCT]
                    if (!cfg) {
                        error("Invalid product selected: ${params.PRODUCT}")
                    }
                    if (!cfg.enabled) {
                        error("Deployment for ${params.PRODUCT} is disabled")
                    }

                    env.TARGET_IP   = cfg.ip
                    env.TARGET_DEST = cfg.dest
                    env.TARGET_CRED = cfg.cred

                    echo "Using credential ID: ${cfg.cred}"
                }
            }
        }

        stage('Validate Local DLL Exists') {
            steps {
                script {
                    def localDllPath = "C:\\Users\\akash.subhransh\\Downloads\\${params.DLL_FILE}"

                    if (!fileExists(localDllPath)) {
                        error("DLL file not found at ${localDllPath}")
                    }

                    env.LOCAL_DLL_PATH = localDllPath
                    echo "DLL found at ${localDllPath}"
                }
            }
        }

        stage('Upload DLL to GCP') {
            steps {
                withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GCP_KEY')]) {
                    bat """
                    gsutil -o Credentials:gs_service_key_file=%GCP_KEY% ^
                    cp "%LOCAL_DLL_PATH%" "${BUCKET}/${params.DLL_FILE}"
                    """
                }
            }
        }

 stage('Generate Signed Download URL') {
    steps {
        withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GCP_KEY')]) {
            script {
                bat """
                gsutil signurl -d 15m "%GCP_KEY%" "${BUCKET}/${params.DLL_FILE}" > signed_url.txt
                """

                def signedOutput = readFile('signed_url.txt').trim()
                def lines = signedOutput.split(/\r?\n/)

                // Skip header line, take data line
                if (lines.size() < 2) {
                    error("Signed URL output is invalid:\\n${signedOutput}")
                }

                // Split by whitespace and take last column (the URL)
                def parts = lines[1].trim().split(/\s+/)
                def signedUrl = parts[-1]

                if (!signedUrl.startsWith("https://")) {
                    error("Failed to extract signed URL:\\n${signedOutput}")
                }

                env.SIGNED_URL = signedUrl

                echo "================ SIGNED DOWNLOAD URL (15 min) ================"
                echo env.SIGNED_URL
                echo "=============================================================="
            }
        }
    }
}


        stage('Download DLL Using Signed URL') {
    steps {
        powershell """
        \$url = '${env.SIGNED_URL}'
        \$out = '${UPLOAD_DIR}\\\\${params.DLL_FILE}'
        Invoke-WebRequest -Uri \$url -OutFile \$out
        """
        echo "DLL downloaded successfully to ${UPLOAD_DIR}\\${params.DLL_FILE}"
    }
}


        stage('Deploy DLL to Server') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: env.TARGET_CRED,
                        usernameVariable: 'DEPLOY_USER',
                        passwordVariable: 'DEPLOY_PASS'
                    )
                ]) {
                    powershell """
                    \$sec  = ConvertTo-SecureString \$env:DEPLOY_PASS -AsPlainText -Force
                    \$cred = New-Object PSCredential(\$env:DEPLOY_USER, \$sec)
                    \$s    = New-PSSession -ComputerName ${env.TARGET_IP} -Credential \$cred

                    Copy-Item "${UPLOAD_DIR}\\${params.DLL_FILE}" "${env.TARGET_DEST}" -ToSession \$s -Force

                    Remove-PSSession \$s
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployment completed successfully"
        }
        failure {
            echo "Deployment failed"
        }
    }
}
