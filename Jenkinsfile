def executeBuildSteps(){
    timestamps {
        echo "Started on: ${new Date()}"

        gitCredentialsId = 'jenkins.niq.windows'
        commonWindowsBuildNode = 'windows&&msvs2010&&msvs2012&&msvs2015'
        artifactsToKeep = 'Source/msbuild.log, Source/Binary/Release/**/*.pdb, Source/Binary/Release/**/*.exe, Source/Binary/Release/**/*.dll, Source/Binary/Release/**/*.xml, Source/**/bin/**/Release/*.dll, Source/**/bin/**/Release/*.exe, Source/RCL.Setup/Output/**, Source/Binary/Release/Tools/MigrationWeb/**, Source/PrecompiledWeb/Web/**'
        buildLabel="${env.JOB_NAME}---${env.BUILD_NUMBER}"
        buildLabel=buildLabel.replace('/','_')
        buildLabel=buildLabel.replace('%2F','_')
        artifactsZip="${buildLabel}-out.zip"

        updatedVersionInFiles = false

        vmTestsSummary = ''
        ciMachineName = '.'
        artifactoryLink = ''
        edgeAdapterArtifactoryLink = ''

        times = [:]
        errors = []
        try
        {
            def initialGitCommitHash
            def ciParams

            echo "Branch name is: ${env.BRANCH_NAME}"

            def startWait = System.currentTimeMillis()
            lock('cj-dv-cbs-v004'){
                def waitDuration = durationAsHumanReadable(System.currentTimeMillis() - startWait);
                times["WAIT-FOR-LOCK"] = waitDuration
                ciMachineName = 'cj-dv-cbs-v004'
                echo "CI machine name  : ${ciMachineName}"

                runOnNodeWithWsCleanup('linux') {
                    runStep('Cleanup UT machine'){

                        //GET (only) the script that ensures the UT machine is "clean" (no UTs are running and the artifacts folder is empty)
                        gitCredentialsId = 'jenkins.niq.windows'
                        sshagent (credentials: [gitCredentialsId]) {
                            def URL="https://dev.azure.com/ngro/Roche%20-%20cobas%20link/_apis/git/repositories/CLINK/archive?format=zip&version=${env.BRANCH_NAME}"
                            sh "curl -u ':jenkins.azure.account' -o j.zip $URL"
                            unzip dir: '.', glob: '', quiet: true, zipFile: 'j.zip'
                            sh 'ls -R'
                        }

                        //TRANSFER the script that ensures the UT machine is "clean" (no UTs are running and the artifacts folder is empty)
                        sshPublisher(
                            publishers: [
                                sshPublisherDesc(
                                    configName: 'cj-dv-cbs-v004',
                                    transfers: [
                                        sshTransfer(
                                            cleanRemote: false,
                                            execTimeout: 120000,
                                            flatten: false,
                                            makeEmptyDirs: false,
                                            noDefaultExcludes: false,
                                            patternSeparator: '[, \n]+',
                                            remoteDirectory: '/', remoteDirectorySDF: false,
                                            sourceFiles: 'Source/CI/**',
                                            excludes: ''
                                        )
                                    ],
                                usePromotionTimestamp: false,
                                useWorkspaceInPromotion: true,
                                verbose: false)
                            ]
                        )

                        //RUN the script that ensures the UT machine is "clean" (no UTs are running and the artifacts folder is empty)
                        sshcred = 'service.cobaslink-unittests.key.credentials'
                        sshagent (credentials: [sshcred]){
                            catchError{
                                sh 'ssh -t -o StrictHostKeyChecking=no cl-jenkins-build@cj-dv-cbs-v004 powershell -file d:\\\\ciroot2\\\\Source\\\\CI\\\\kill-running-CITasks.ps1'
                            }
                        }
                    }
                }

                runOnNodeWithWsCleanup(commonWindowsBuildNode) {
                    runStage("Cleanup"){
                        step([$class: 'WsCleanup'])
                    }

                    runStage("Checkout"){
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: '**']],
                            doGenerateSubmoduleConfigurations: false,
                            extensions: [[
                                    $class: 'CloneOption',
                                    depth: 100,
                                    honorRefspec: true,
                                    noTags: true,
                                    reference: '',
                                    shallow: true
                            ]],
                            submoduleCfg: [],
                            userRemoteConfigs: [[
                                credentialsId: 'jenkins.niq.windows',
                                refspec: "+refs/heads/${env.BRANCH_NAME}:refs/remotes/origin/${env.BRANCH_NAME}",
                                url: 'git@ssh.dev.azure.com:v3/ngro/Roche%20-%20cobas%20link/CLINK'
                            ]]
                        ])

                        buildAuthor = bat returnStdout: true, script: '@git log -1 --format=\'%%aN\''
                        buildAuthor = buildAuthor.replace('\'','').replace('\n','')
                        NotifyStash()

                        initialGitCommitHash = bat(returnStdout: true, script: 'git rev-parse HEAD').split('\n')[2].trim()
                        ciParams = readJSON file: 'Source/CI/CI-params.json'
                    }

                    runStage('Build'){
                        tool 'MSBuild Community Tasks'
                        String msBuild = "${tool 'MSBuild v17'}"
                        runStep("Build BuildTools.sln"){
                            dir('Source/Tools/Building') {
                                def bstat = bat returnStatus:true, script: "\"${msBuild}\" -t:restore BuildTools.sln -v:m"
                                if (bstat != 0){
                                    error 'Restoring Nuget packages for "BuildTools.sln" failed!'
                                }
                                bstat = bat returnStatus:true, script: "\"${msBuild}\" BuildTools.sln /p:Configuration=Release -v:m"
                                if (bstat != 0){
                                    error 'Building "BuildTools.sln" failed!'
                                }
                            }
                        }

                        runStep("Build build-jenkins.proj"){
                            dir('Source') {
                                withEnv(["RocheRIGTasksPath=${pwd()}\\Tools\\Building\\Roche.CLink.Tools.Build.MsBuild\\bin\\Release",
                                        "BuildConfiguration=Release",
                                        "BuildVersionType=Client", //force updating the version
                                        "DoLocalFilesUpdate=true",
                                        "BuildDeliverables=true",
                                        "PATH=${env.PATH};${pwd()}\\Tools\\Building\\Inno", //so we don't install Inno on the build machine
                                        "VersionedFilesOutFile=${ciParams.versionedFilesOutFile}",
                                        "RCLVersion=${ciParams.RCLversion}",
                                        "_MSPDBSRV_ENDPOINT_=${env.BUILD_TAG}"]) {
                                    def bstat = bat returnStatus:true, script:"\"${msBuild}\" /fl /clp:NoSummary;ShowTimestamp build-jenkins.proj /p:Configuration=%BuildConfiguration% /p:BuildVersionType=%BuildVersionType% /p:PlatformName=\"Mixed Platforms\" /p:DoLocalFilesUpdate=%DoLocalFilesUpdate% /p:BuildDeliverables=%BuildDeliverables% /p:VersionedFilesOutFile=%VersionedFilesOutFile% /p:RCLVersion=%RCLVersion% -v:m"
                                    if (bstat != 0){
                                        error 'Building "build-jenkins.proj" failed!'
                                    }

                                    def vf = readFile "${ciParams.versionedFilesOutFile}"
                                    updatedVersionInFiles =  vf != ''
                                }
                            }

                            runStage("Archive Artifacts"){
                                zip zipFile: "Source/${artifactsZip}", glob: artifactsToKeep
                            }

                            if (!updatedVersionInFiles){
                                runStep("Transfer artifacts"){
                                        sshPublisher(publishers: [sshPublisherDesc(configName: 'cj-dv-cbs-v004', transfers: [sshTransfer(
                                            cleanRemote: false,
                                            execTimeout: 120000,
                                            flatten: false,
                                            makeEmptyDirs: false,
                                            noDefaultExcludes: false,
                                            patternSeparator: '[, \n]+',
                                            remoteDirectory: '/', remoteDirectorySDF: false,
                                            sourceFiles: """
                                                            Source/*-out.zip,
                                                            Source/CI/**,
                                                            Source/RCL.Setup/**/Output/**,
                                                            Source/RCL.Setup/bin/Neutral.Installer.WinApp.exe,
                                                            Source/Registry/*.reg,
                                                            Source/**/*.sql,
                                                            Source/ModularDriver/rigm_for_SmartComDriver.xsl,
                                                            Source/ModularDriver/SmartComIPC_2_RIGM.xsl,
                                                            Source/SharedBinaries/Roche.TestDocGen.Converter/**,
                                                            Source/ComCryptography/bin/Release/ComCryptography.dll,
                                                            Source/Binary/Release/Roche.RIG.CLink.API.dll,
                                                            Source/cobaselibrary.API/UnitTests/bin/Release/**,
                                                            Source/CoreSystem.UnitTest/bin/Release/**,
                                                            Source/DataManager.UnitTests/bin/Release/**,
                                                            Source/InventoryDatabase.UnitTest/bin/Release/**,
                                                            Source/Ipc/Ipc.UnitTests/bin/Release/**,
                                                            Source/IpcMessages/IpcMessages.UnitTests/bin/Release/**,
                                                            Source/Logging/Logging.Tests/bin/Release/**,
                                                            Source/ManagementConsole.UnitTests/bin/Release/**,
                                                            Source/Messaging/Messaging.Tests/bin/Release/**,
                                                            Source/ModularDriver.UnitTest/bin/Release/**,
                                                            Source/PackageManagement/PackageManagementLogic.UnitTests/bin/Release/**,
                                                            Source/Roche.RIG.CLink.Upload.UnitTests/bin/Release/**,
                                                            Source/Roche2Axeda.UnitTest/bin/Release/**,
                                                            Source/RSI/Tests/RSI.UnitTests/bin/Release/**,
                                                            Source/RSI2/UnitTests/bin/Release/**,
                                                            Source/SQLServerAgent/Roche.RIG.CLink.SQLServerAgent.UnitTests/bin/Release/**,
                                                            Source/StandardLibraries/Roche.RIG.CLink.CDC.UnitTest/bin/Release/**,
                                                            Source/StandardLibraries/Roche.RIG.CLink.Common.Metadata.UnitTest/bin/Release/**,
                                                            Source/SwidDownloaderService/UnitTests/bin/Release/**,
                                                            Source/SystemAudit.Tsn/SystemAudit.Tsn.UnitTests/bin/Release/**,
                                                            Source/SystemAudit.UnitTests/bin/Release/**,
                                                            Source/SystemCheck.UnitTests/bin/Release/**,
                                                            Source/Tools/SmartCom/AssetsSupportTool/AssetsSupportTool.Tests/bin/Release/**,
                                                            Source/Tools/SmartCom/ParameterDeleteTool/ParameterDeleteTool.Tests/bin/Release/**,
                                                            Source/Tools/SmartCom/PrinterTool/PrinterTool.UnitTests/bin/Release/**,
                                                            Source/EdgeAdapter/Tests/Business.IntegrationTests/bin/Release/**,
                                                            Source/EdgeAdapter/Tests/Business.Tests/bin/Release/**,
                                                            Source/EdgeAdapter/Tests/Support.Tests/bin/Release/**
                                                        """,
                                            excludes: ''
                                        )],
                                        usePromotionTimestamp: false,
                                        useWorkspaceInPromotion: true,
                                        verbose: false)])
                                }
                            }

                            runStep('Stash needed files'){
                                powershell script: '''
                                                # The build process, produces some changes to certain files that contain versions.
                                                # Their filenames are captured in the file $ciParams.versionedFilesOutFile.
                                                # Here we read from that file and produce a stash that contains the modified files.
                                                # The stash will later be used on a linux node (due to some technical issues on windows
                                                # nodes) and the files are commited to git.
                                                $baseDir = Get-Location
                                                $baseDir = [System.IO.Path]::GetFullPath($baseDir) + "\\"
                                                $vfiles = (Get-content -Path "Source/versioned-files.txt") -split "\n" |
                                                            Where-Object { $_ -and $_.trim() -ne "" } |
                                                            ForEach-Object { $_ -replace [regex]::Escape($baseDir), "" -replace "\\\\", "/" } |
                                                            Where-Object { $_ -and $_.trim() -ne "" }
                                                $vfiles -join ", "  | Out-File "stash-files.txt" -Force -noNewLine -Encoding ASCII
                                '''
                                stashFiles = readFile "stash-files.txt"
                                stashFiles = stashFiles + ',Source/versioned-files.txt,Source/CI/msbuild-log-parser.rules,Source/RCL.Setup/Output/**,Source/Binary/Release/SPITLot.exe,Source/Binary/Release/SPITGen.exe,Source/Binary/Release/SPITGenCumulativeUpdate.exe,Source/Binary/Release/AssetsSupportToolCompiled/assets.xml'
                                println "Stashing: ${stashFiles}"
                                stash name: "Needed files", includes: stashFiles, useDefaultExcludes: false
                            }
                        }
                    }
                }//end runOnNodeWithWsCleanup(commonWindowsBuildNode) - BUILD

                if (!updatedVersionInFiles){
                    runOnNodeWithWsCleanup('linux') {

                        if (buildSettings["[CI] Run Unit Tests"]){
                            sshcred = 'service.cobaslink-unittests.key.credentials'
                            sshagent (credentials: [sshcred]){

                            // parallel (
                            //     runUnitTests: {
                                    runStage("[CI] Run Unit Tests"){
                                        scr = "ssh -t -o StrictHostKeyChecking=no cl-jenkins-build@cj-dv-cbs-v004 powershell -file d:\\\\ciroot2\\\\Source\\\\CI\\\\Run-CITasks.ps1 -localRootPath d:\\\\ciroot2\\\\Source -phase all -settingsFile d:\\\\ciroot2\\\\Source\\\\ci\\\\CI-params.json -artifacts ${artifactsZip} -buildLabel ${buildLabel}"
                                        def bstat = sh returnStdOut:true, returnStatus:true, script: scr
                                        if (bstat != 0){
                                            error 'Error running CI tasks!'
                                        }
                                    }
                            //     },
                            //     runVmTests: {
                            //         runStage("[CI] Run cobas link Tests"){
                            //             catchError{
                            //                 startWait = System.currentTimeMillis()
                            //                 lock(resource: 'cl-tests'){
                            //                     waitDuration = durationAsHumanReadable(System.currentTimeMillis() - startWait);
                            //                     times["WAIT-FOR-LOCK-CL"] = waitDuration
                            //                     runCIphase "runCobasLinkTests"
                            //                 }
                            //             }
                            //         }
                            //     }
                            // )//end parallel

                                runStage("[CI] Collect results"){

                                    runStep('Copy test results'){
                                        dir('Source'){
                                            sh 'mkdir UTs-Results'
                                            sh 'scp -r -o StrictHostKeyChecking=no cl-jenkins-build@cj-dv-cbs-v004:/UTs-Results/{TestResults.zip,*.trx,*.xml} ./UTs-Results'
                                        }
                                    }

                                    runStep('Record unit test results'){
                                        dir('Source'){
                                            def utResultFiles = findFiles(glob: 'UTs-Results/*.xml')
                                            println "Got ${utResultFiles.size()} out of ${ciParams.unitTestContainers.size()} test results files"

                                            for(ii =0; ii<utResultFiles.size(); ii++)
                                            {
                                                def utResult = utResultFiles[ii]
                                                println utResult.path
                                                junit "${utResult.path}"
                                            }

                                            if(ciParams.unitTestContainers.size() != utResultFiles.size()){
                                                error 'Not all test results were found!'
                                            }
                                        }
                                    }

                                    runStep('Record vm tests results'){
                                        dir('Source'){
                                            def vmTestsSummaryFiles = findFiles(glob: 'UTs-Results/vmTestsSummary.txt')
                                            println "Got ${vmTestsSummaryFiles.size()} vmTestsSummary files"
                                            for(ii =0; ii<vmTestsSummaryFiles.size(); ii++)
                                            {
                                                def summary = vmTestsSummaryFiles[ii]
                                                println summary.path
                                                vmTestsSummary = readFile summary.path
                                            }
                                        }
                                    }
                                }
                            }//sshagent

                            if(env.BRANCH_NAME == 'master' || env.BRANCH_NAME.startsWith('dev-test-azure-devops')) {
                                checkout([
                                    $class: 'GitSCM',
                                    branches: [[name: '**']],
                                    doGenerateSubmoduleConfigurations: false,
                                    extensions: [[
                                            $class: 'CloneOption',
                                            depth: 500, //max no of commits
                                            honorRefspec: true,
                                            noTags: true,
                                            reference: '',
                                            shallow: true
                                    ]],
                                    submoduleCfg: [],
                                    userRemoteConfigs: [[
                                        credentialsId: 'jenkins.niq.windows',
                                        refspec: "+refs/heads/${env.BRANCH_NAME}:refs/remotes/origin/${env.BRANCH_NAME}",
                                        url: 'git@ssh.dev.azure.com:v3/ngro/Roche%20-%20cobas%20link/CLINK'
                                    ]]
                                ])

                                runStep("Extracting change log..."){
                                    sh (script: 'git --no-pager log '+initialGitCommitHash+' --format=format:\'[%aI] [%an] %B\' --after=\'2021-11-01\'  --invert-grep  --grep=\'\\[Build\\]Update versions for \' > changelog.txt', returnStdout: true)
                                }

                                runStep('[linux] Unstash'){
                                    unstash name: "Needed files"
                                }

                                sshcred = 'service.cobaslink-unittests.key.credentials'
                                sshagent (credentials: [sshcred]){
                                    runStep('Copy test results'){
                                        dir('Source'){
                                            sh 'mkdir -p UTs-Results'
                                            sh 'scp -r -o StrictHostKeyChecking=no cl-jenkins-build@cj-dv-cbs-v004:/UTs-Results/TestResults.zip ./UTs-Results'
                                        }
                                    }
                                }

                                runStep("Upload to Artifactory"){
                                    echo "Identifying output install kit..."
                                    def kit = findFiles(glob: 'Source/RCL.Setup/Output/RCL-Update-*.exe')[0]
                                    println "Found output install kit: [${kit.path}]"

                                    echo "Uploading to Artifactory..."
                                    server = Artifactory.newServer url: 'https://repo.nagarro.com', credentialsId: 'service.jenkins.artifactory.token'
                                    String crtDate = new java.text.SimpleDateFormat("yyyyMMdd.HHmmss").format(new java.util.Date())
                                    String uploadSpec = """
                                        {
                                            "files": [
                                                {
                                                    "pattern": "${kit.path}",
                                                    "target" : "clnk/RCL/${env.BRANCH_NAME}/${crtDate}-${initialGitCommitHash}/${kit.name}"
                                                },
                                                {
                                                    "pattern": "changelog.txt",
                                                    "target" : "clnk/RCL/${env.BRANCH_NAME}/${crtDate}-${initialGitCommitHash}/changelog.txt"
                                                },
                                                {
                                                    "pattern": "Source/UTs-Results/TestResults.zip",
                                                    "target" : "clnk/RCL/${env.BRANCH_NAME}/${crtDate}-${initialGitCommitHash}/TestResults.zip"
                                                },
                                                {
                                                    "pattern": "Source/Binary/Release/SPITLot.exe",
                                                    "target" : "clnk/RCL/${env.BRANCH_NAME}/${crtDate}-${initialGitCommitHash}/Tools/SPITLot.exe"
                                                },
                                                {
                                                    "pattern": "Source/Binary/Release/SPITGen.exe",
                                                    "target" : "clnk/RCL/${env.BRANCH_NAME}/${crtDate}-${initialGitCommitHash}/Tools/SPITGen.exe"
                                                },
                                                {
                                                    "pattern": "Source/Binary/Release/SPITGenCumulativeUpdate.exe",
                                                    "target" : "clnk/RCL/${env.BRANCH_NAME}/${crtDate}-${initialGitCommitHash}/Tools/SPITGenCumulativeUpdate.exe"
                                                }
                                            ]
                                        }"""
                                    server.upload(uploadSpec)

                                    artifactoryLink = """
                                    <tr>
                                        <td>Artifactory location</td>
                                        <td>: <a href="https://repo.nagarro.com/clnk/RCL/${env.BRANCH_NAME}/${crtDate}-${initialGitCommitHash}">here</a>
                                        </td>
                                    </tr>"""
                                }
                            }
                        }//end run unittests
                    }//end node
                }
            }//end lock

            if (buildSettings["Build"]){
                runOnNodeWithWsCleanup('linux'){
                    def files
                    runStep('[linux] Unstash'){
                        unstash name: "Needed files"
                    }
                    dir('Source') {
                        def f = readFile ciParams.versionedFilesOutFile
                        files = f.replace("\\", '/').split("\r\n").join(" ")
                        echo files
                    }
                    if (files.size()>0){
                        stage("Commit versioned files"){
                            step([$class: 'WsCleanup'])

                            runStep('[linux] Checkout'){
                                checkout([
                                    $class: 'GitSCM',
                                    branches: [[name: '**']],
                                    doGenerateSubmoduleConfigurations: false,
                                    extensions: [[
                                            $class: 'CloneOption',
                                            depth: 100,
                                            honorRefspec: true,
                                            noTags: true,
                                            reference: '',
                                            shallow: true
                                    ]],
                                    submoduleCfg: [],
                                    userRemoteConfigs: [[
                                        credentialsId: 'jenkins.niq.windows',
                                        refspec: "+refs/heads/${env.BRANCH_NAME}:refs/remotes/origin/${env.BRANCH_NAME}",
                                        url: 'git@ssh.dev.azure.com:v3/ngro/Roche%20-%20cobas%20link/CLINK'
                                    ]]
                                ])
                                currentGitCommitHash = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()

                                if (initialGitCommitHash != currentGitCommitHash){
                                    error "Changes detected on the repository between original checkout and checkin time! [${initialGitCommitHash}  vs  ${currentGitCommitHash}]"
                                }
                            }

                            runStep('[linux] Unstash'){
                                unstash name: "Needed files"
                            }

                            runStep('Commit versioned files'){
                                sshagent (credentials: [gitCredentialsId]) {
                                    String shScript = """
                                        git config core.autocrlf false
                                        git config user.name "jenkins-build"
                                        git config user.email jenkins@nagarro.com
                                        git add .
                                        git status
                                        if git commit -m \"[Build] Update versions for ${ciParams.RCLversion}\"; then
                                            git push --set-upstream origin HEAD:${env.BRANCH_NAME};
                                        fi
                                    """
                                    sh shScript
                                }
                            }//end step
                        }//stage
                    }
                    else{
                        echo "No file version has been updated"
                    }
                }//node linux
            }//if
        }
        catch(e){
            currentBuild.result = "FAILURE"
            echo "########### ERROR caught! ${e}"
            errors.add("${e}")
            throw e
        }
        finally{
            printTimes()
            try{
                runOnNodeWithWsCleanup(commonWindowsBuildNode){
                    try{
                        runStep('[Windows] Unstash'){
                            unstash name: "Needed files"
                        }
                    }
                    catch(e){
                        echo "########### ERROR caught! ${e}"
                    }
                    runStep('Parse log'){
                        echo "calling parse log using: ${pwd()}\\Source\\CI\\msbuild-log-parser.rules"
                        step([$class: 'LogParserPublisher', failBuildOnError: true, projectRulePath: "${pwd()}\\Source\\CI\\msbuild-log-parser.rules", showGraphs: true, useProjectRule: true])

                        currentBuild.rawBuild.getLog(Integer.MAX_VALUE).each() { line ->
                            def m = line =~ /(.*: error .*:.*)/
                            if (m.find()) {
                                errors.add(m.group())
                            }
                        }
                    }
                }
            }
            finally{
                sendEmail()

                runOnNodeWithWsCleanup(commonWindowsBuildNode){
                    def slackMessage = "Finished build\n> Triggered by *[${buildAuthor}]*"
                    def slackColor = 'danger'
                    def buildStatusEmotion = ':-1:'
                    if (currentBuild.currentResult == 'SUCCESS') {
                        slackColor = 'good'
                        buildStatusEmotion = ':+1:'
                    }

                    script {
                        currentBuild.result = currentBuild.result ?: 'SUCCESS'
                        NotifyStash()
                    }
                    echo getErrorsSummary()
                }
            }
        }
    }//timestamps
} //executeBuildSteps

def runStage(stageName, def stageBlock){
    if (buildSettings[stageName]){ //only if the stage is enabled
        def start = System.currentTimeMillis()
        echo ">>>>>>>>>>> [START] ${stageName}"
        stage(stageName){
            stageBlock();
        }
        def d = System.currentTimeMillis() - start
        def hrduration = durationAsHumanReadable(d);
        println "<<<<<<<<<<< [END] ${stageName} (Duration: ${hrduration})"
        times[stageName]=hrduration
    }
}

def runStep(stepName, def stepBlock){
    def start = System.currentTimeMillis()
    echo ">>>>>>>>>>> [START] ${stepName}"
    stepBlock();
    def d = System.currentTimeMillis() - start
    def hrduration = durationAsHumanReadable(d);
    println "<<<<<<<<<<< [END] ${stepName} (Duration: ${hrduration})"
    times[stepName]=hrduration
}

def durationAsHumanReadable(millis){
    def stdout;
    def duration;
    if (env.NODE_NAME == null){
        node('linux'){
            stdout = sh script: "python -c\"import datetime;print datetime.timedelta(milliseconds=${millis});\"", returnStdout: true
        }
        duration = (stdout =~ /(.*[0-9]+:[0-9][0-9]:[0-9][0-9]).*/)[0][1]
    }
    else if (env.NODE_NAME.indexOf('win') < 0){
        stdout = sh script: "python -c\"import datetime;print datetime.timedelta(milliseconds=${millis});\"", returnStdout: true
        duration = (stdout =~ /(.*[0-9]+:[0-9][0-9]:[0-9][0-9]).*/)[0][1]
    }
    else{
        stdout = bat script: "python -c\"import datetime;print datetime.timedelta(milliseconds=${millis});\"", returnStdout: true
        duration = (stdout =~ /.*\n(.*[0-9]+:[0-9][0-9]:[0-9][0-9]).*/)[0][1]
    }
    return duration
}

def runOnNodeWithWsCleanup(nodeName, def scriptBlock){
    node(nodeName){
        try{
            scriptBlock()
        }
        finally{
            deleteDir()
        }
    }
}

def getTimesSummary(){
    def timesSummary = "The following durations have been recorded during this build:"
    for(e in times){
        timesSummary += "\n>>> ${e.key}: ${e.value}"
    }
    return timesSummary
}

def getErrorsSummary(){
    def errorsSummary = "The following errors occurred during this build:"
    for(e in errors){
        errorsSummary += "\n>>> ${e}"
    }
    return errorsSummary

}

def printTimes(){
    echo getTimesSummary()
}

def sendEmail(){
    def resultColor = ""
    def green = "#33BB66"
    def red = "#FF0000"
    if (currentBuild.currentResult == "SUCCESS"){
        resultColor = green
    }
    else{
        resultColor = red
    }

    emailext (
        to: defaultEmailTOs,
        subject: "${currentBuild.currentResult}: build [${env.BUILD_NUMBER}] - [${env.JOB_NAME.replace('%2F','/')}]",
        recipientProviders: [[$class: 'DevelopersRecipientProvider'], [$class: 'CulpritsRecipientProvider'], [$class: 'RequesterRecipientProvider']],
        mimeType: 'text/html',
        body: """<table>
              <tr>
                <td>Job</td>
                <td>: <a href="\${PROJECT_URL}">\${PROJECT_NAME}</a></td>
              </tr>
              <tr>
                <td>Build number</td>
                <td>: \${BUILD_NUMBER}</td>
              </tr>
              <tr>
                <td>Current branch</td>
                <td>: \${PROJECT_DISPLAY_NAME}</td>
              </tr>
              ${artifactoryLink}
              ${edgeAdapterArtifactoryLink}
              <tr>
                <td>Drop folder</td>
                <td>: \\\\${ciMachineName}\\BuildArtifacts\\${buildLabel}</td>
              </tr>
              <tr>
                <td>Build status</td>
                <td>: \${BUILD_STATUS}</td>
              </tr>
              <tr>
                <td>Result</td>
                <td>: <span style=\"color:${resultColor}\">${currentBuild.currentResult}</span></td>
              </tr>
              <tr>
                <td>Cause</td>
                <td>: \${BUILD_CAUSE}</td>
              </tr>
              <tr>
                <td>Duration</td>
                <td>: ${currentBuild.durationString.replace(' and counting', '')}</td>
              </tr>
              <tr>
                <td>Build errors</td>
                <td>
                    <table style="font-family: courier new; font-size: 11">
                    ${getErrorsSummary().replace('>>>','</td></tr><tr><td>')}
                    </td></tr></table>
                </td>
              </tr>
              <tr>
                <td>Unit Tests</td>
                <td>: <span style="color:${green}">\${TEST_COUNTS, var="pass"}</span> : <span style="color:${red}">\${TEST_COUNTS, var="fail"}</span> (out of <strong>\${TEST_COUNTS,var="total"}</strong>)</td>
              </tr>
              ${vmTestsSummary}
              <tr>
                <td>Failed tests</td>
                <td>: \${FAILED_TESTS, maxTests=25, showStack=false, showMessage=true} </td>
              </tr>
            </table>
            <p>Changes since last success
                <ul>
                \${CHANGES_SINCE_LAST_SUCCESS, format="<li>Build #%n<ul>%c</ul></li>", changesFormat="<li>[%d]<b> %a</b>: %m</li>"}
                </ul>
            </p>
            <p>
                <table style="font-family: courier new; font-size: 11">
                ${getTimesSummary().replace('>>>','</td></tr><tr><td>').replace(': ', '</td><td align="right">')}
                </td></tr></table>
            </p>
            """
      )
}

def NotifyStash() {
    step([$class: 'StashNotifier', commitSha1: '', credentialsId: 'service.jenkins.ad.credentials', disableInprogressNotification: false, ignoreUnverifiedSSLPeer: false, includeBuildNumberInKey: false, prependParentProjectKey: false, projectKey: '', stashServerBaseUrl: 'https://git-niq.nagarro.com/'])
}

// !!Important Boilerplate!!
// The external code must return it's contents as an object
return this;
