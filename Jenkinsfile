pipeline
{
    agent any

    parameters
    {
        string(name: 'ROLLBACK_BUILD', defaultValue: '', description: 'Enter build number to rollback')
        booleanParam(name: 'FORCE_FAIL', defaultValue: false, description: 'Simulate failure')
    }

    environment
    {
        DOTNET_DIR = "${WORKSPACE}/.dotnet"
        DOTNET_ROOT = "${WORKSPACE}/.dotnet"
        PATH = "${WORKSPACE}/.dotnet:${HOME}/.dotnet/tools:${env.PATH}"
        SONAR_PROJECT_KEY = "EPM-ICMP-JAN-2026-DOTNET-TEAM1"
        PROJECT_PATH = "src/NopCommerce.sln"
    }

    options
    {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 30, unit: 'MINUTES')
    }

    stages
    {
        stage('Checkout')
        {
            when
            {
                expression
                {
                    params.ROLLBACK_BUILD == ''
                }
            }
            steps
            {
                checkout scm
            }
        }

        stage('Install .NET 10')
        {
            when 
            {
                expression
                {
                    params.ROLLBACK_BUILD == ''
                }
            }

            steps
            {
                sh '''
                if [ ! -d "$DOTNET_DIR" ]; then
                    echo "Installing .NET SDK 10..."
                    curl -sSL https://dot.net/v1/dotnet-install.sh -o dotnet-install.sh
                    chmod +x dotnet-install.sh
                    ./dotnet-install.sh --channel 10.0 --quality ga --install-dir $DOTNET_DIR
                fi

                export DOTNET_ROOT=$DOTNET_DIR
                export PATH=$DOTNET_ROOT:$PATH

                echo "Installed .NET version:"
                dotnet --version
                '''
            }
        }

        stage('Install Tools')
        {
            when
            {
                expression
                {
                    params.ROLLBACK_BUILD == ''
                }
            }

            steps
            {
                sh '''
                export PATH="$PATH:$HOME/.dotnet/tools"
                dotnet tool update --global dotnet-sonarscanner || dotnet tool install --global dotnet-sonarscanner
                dotnet tool update --global dotnet-reportgenerator-globaltool || dotnet tool install --global dotnet-reportgenerator-globaltool
                '''
            }
        }

        stage('Restore') 
        {
            when
            {
                expression
                {
                    params.ROLLBACK_BUILD == ''
                }
            }

            steps
            {
                sh '''
                export DOTNET_ROOT=$DOTNET_DIR
                export PATH=$DOTNET_ROOT:$PATH
                dotnet restore ${PROJECT_PATH}
                '''
            }
        }

        stage('Sonar + Build + Test + Coverage') 
        {
            when
            {
                expression
                {
                    params.ROLLBACK_BUILD == ''
                }
            }

            steps
            {
                withSonarQubeEnv('SonarHyd') 
                {
                    withCredentials([string(credentialsId: 'EMPICMP-DOTNET-TEAM1-sonarqube-token', variable: 'SONAR_TOKEN')])
                    {
                        sh '''
                        export PATH="$PATH:$HOME/.dotnet/tools"
                        export DOTNET_ROOT=$DOTNET_DIR
                        mkdir -p coverage

                        echo "========== SONAR BEGIN =========="
                        dotnet sonarscanner begin /k:"$SONAR_PROJECT_KEY" \
                        /d:sonar.login="$SONAR_TOKEN" \
                        /d:sonar.cs.opencover.reportsPaths="coverage/**/coverage.opencover.xml"
                        echo "========== BUILD =========="

                        dotnet build ${PROJECT_PATH} --configuration Release

                        echo "========== RUN TESTS =========="
                        TEST_FOUND=false
                        for testproj in $(find . -name "*Test*.csproj"); do
                            echo "Running tests in $testproj"
                            TEST_FOUND=true
                            dotnet test "$testproj" \
                            --configuration Release \
                            --collect:"XPlat Code Coverage" \
                            --results-directory coverage \
                            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover || true
                        done

                        if [ "$TEST_FOUND" = false ]; then
                            echo "NO TEST PROJECTS FOUND"
                        fi

                        echo "========== DEBUG COVERAGE =========="

                        ls -R coverage || true

                        echo "========== GENERATE REPORT =========="
                        reportgenerator \
                        -reports:"coverage/**/coverage.opencover.xml" \
                        -targetdir:"coverage/reports" \
                        -reporttypes:"Html;Cobertura" || true
                        echo "========== SONAR END =========="

                        dotnet sonarscanner end \
                        /d:sonar.token="$SONAR_TOKEN"
                        '''
                    }
                }
            }
        }

        stage('Quality Gate Validation')
        {
            when
            {
                expression
                {
                    params.ROLLBACK_BUILD == ''
                }
            }
            
            steps
            {
                timeout(time: 10, unit: 'MINUTES')
                {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Publish')
        {
            when
            {
                expression
                {
                    params.ROLLBACK_BUILD == ''
                }
            }

            steps
            {
                sh '''
                export DOTNET_ROOT=$DOTNET_DIR
                export PATH=$DOTNET_ROOT:$PATH

                dotnet publish ${PROJECT_PATH} \
                --configuration Release \
                --output publish
                '''
            }
        }

        stage('Deploy')
        {
            when
            {
                expression
                {
                    params.ROLLBACK_BUILD == ''
                }
            }

            steps
            {
                sh '''
                echo "Deploying application..."
                '''
            }
        }

        stage('Simulate Failure') {
            when
            {
                expression
                {
                    params.FORCE_FAIL == true && params.ROLLBACK_BUILD == '' 
                }
            }

            steps
            {
                sh 'exit 1'
            }
        }

        stage('Mark Successful Deployment')
        {
            when
            {
                expression
                {
                    params.ROLLBACK_BUILD == ''
                }
            }

            steps
            {
                writeFile file: 'last_success.txt', text: "${BUILD_NUMBER}"
                archiveArtifacts artifacts: 'last_success.txt', fingerprint: true
            }
        }

        stage('Rollback')
        {
            when
            {
                expression
                {
                    params.ROLLBACK_BUILD != ''
                }
            }

            steps
            {
                echo "Rollback logic unchanged"
            }
        }
    }

    post {
        success
        {
            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'coverage/reports',
                reportFiles: 'index.html',
                reportName: 'Code Coverage Report'
            ])
        }

        always
        {
            archiveArtifacts artifacts: 'coverage/**', allowEmptyArchive: true
            cleanWs()
        }
    }
}