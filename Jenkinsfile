pipeline {
    agent any

    /*
     * Parameters:
     * PRODUCT       -> Target server selection
     * UPLOAD_FILE   -> DLL uploaded from local system (Pipeline-safe)
     */
    parameters {

        // Product selection
        choice(
            name: 'PRODUCT',
            choices: ['Agoda', 'Booking'],
            description: 'Select target product/server'
        )

        // Pipeline-compatible file upload parameter
        stashedFile(
            name: 'UPLOAD_FILE',
            description: 'Upload a fresh .dll file from your local system'
        )
    }

    stages {

        /*
         * Resolve server configuration based on selected product
         */
        stage('Resolve Product Configuration') {
            steps {
                script {
                    def products = [
                        'Agoda': [
                            server: '10.10.10.54',
                            dest  : 'C:\\RankBot\\agoda',
                            cred  : 'agoda-creds'
                        ],
                        'Booking': [
                            server: '10.10.10.63',
                            dest  : 'C:\\RankBot\\booking',
                            cred  : 'booking-creds'
                        ]
                    ]

                    def cfg = products[params.PRODUCT]

                    if (!cfg) {
                        error "No configuration found for selected product"
                    }

                    // Save cfg for later stages
                    env.TARGET_SERVER = cfg.server
                    env.TARGET_DEST   = cfg.dest
                    env.TARGET_CRED   = cfg.cred
                }
            }
        }

        /*
         * Confirm Uploaded DLL and prepare in workspace
         */
        stage('Confirm File Upload and Path') {
    steps {
        script {
            // Unstash the uploaded file into the workspace
            unstash 'UPLOAD_FILE'

            // Determine the final DLL filename
            def finalFileName = env.UPLOAD_FILE_FILENAME ?: 'uploaded.dll'

            // Delete old file if it exists to avoid rename error
            bat """
                if exist ${finalFileName} del /F /Q ${finalFileName}
            """

            // Rename the uploaded file
            bat """
                if exist UPLOAD_FILE ren UPLOAD_FILE ${finalFileName}
            """

            // Construct the full path
            def finalFilePath = "${env.WORKSPACE}\\${finalFileName}"
            env.FINAL_DLL_PATH = finalFilePath

            // Verify the renamed file exists
            if (!fileExists(finalFileName)) {
                error "Uploaded file not found after renaming"
            }

            echo "File uploaded successfully"
            echo "File stored at Jenkins workspace path:"
            echo finalFilePath
            echo "Congratulations! File upload completed successfully. Proceeding for next stage."
        }
    }
}


        /*
         * Deploy DLL to selected target server
         */
        stage('Deploy DLL to Server') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: env.TARGET_CRED,
                            usernameVariable: 'DEPLOY_USER',
                            passwordVariable: 'DEPLOY_PASS'
                        )
                    ]) {
                        powershell """
                        \$sec = ConvertTo-SecureString \$env:DEPLOY_PASS -AsPlainText -Force
                        \$cred = New-Object System.Management.Automation.PSCredential(\$env:DEPLOY_USER, \$sec)

                        \$session = New-PSSession -ComputerName ${env.TARGET_SERVER} -Credential \$cred

                        Copy-Item `
                            "\$env:FINAL_DLL_PATH" `
                            "${env.TARGET_DEST}" `
                            -ToSession \$session `
                            -Force

                        Remove-PSSession \$session
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Deployment completed successfully'
        }
        failure {
            echo 'Deployment failed'
        }
        always {
            echo 'Pipeline execution finished'
        }
    }
}
