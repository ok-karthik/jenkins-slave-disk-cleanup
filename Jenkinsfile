import hudson.plugins.sshslaves.SSHLauncher;
idleSlaves = []
busySlaves = []
doneSlaves = []
skipSlaves = []

def getNodeNamesWithZeroBuilds(label) 
{
    def allNodes = Jenkins.getInstance().getLabel(label).getNodes()
    for (int i =0; i < allNodes.size(); i++) 
    {
        Slave node = allNodes[i]
        def launcher = node.getLauncher()

        if (node.getComputer().countBusy()==0 && (launcher instanceof SSHLauncher))
        {
            idleSlaves.add(node.name)
        }
    }
}

def getNodeNamesWithRunningBuilds(label) 
{
    def allNodes = Jenkins.getInstance().getLabel(label).getNodes()
    for (int i =0; i < allNodes.size(); i++)
    {
        Slave node = allNodes[i]
        def launcher = node.getLauncher()

        if (node.getComputer().countBusy()>0 && (launcher instanceof SSHLauncher))
        {
            busySlaves.add(node.name)
        }
    }
}

def getNumberOfRunningBuilds(slave_name)
{
    return Jenkins.instance.nodes.find({it.name == slave_name}).getComputer().countBusy()
}

def isOffline(slave_name)
{
    return Jenkins.instance.nodes.find({it.name == slave_name}).getComputer().isOffline()
}

def passwordlessSSHTest(slave_dns)
{
    ssh_test_cmd = "ssh -o PasswordAuthentication=no " + slave_dns + " exit"
    return sh (returnStatus: true, script: ssh_test_cmd)
}

def markSlaveOffline(slave_name)
{
    Jenkins.instance.nodes.find({it.name == slave_name}).computer.setTemporarilyOffline(true, new hudson.slaves.OfflineCause.ByCLI("disk cleanup"))
    return "${slave_name} marked OFFLINE"
}

def bringSlaveOnline(slave_name)
{
    Jenkins.instance.nodes.find({it.name == slave_name}).computer.setTemporarilyOffline(false, null)
    return "${slave_name} brought back ONLINE"
}

def getWorkspaceLocation(slave_name)
{
    return Jenkins.instance.nodes.find({it.name == slave_name}).getRemoteFS()
}

def executeCmdOnSlave(slave_dns, cmd)
{
    sh """
        set +x
        ssh -o StrictHostKeyChecking=no -A jenkins@$slave_dns "${cmd}"
    """
}

def getDiskUsedPercentage(slave_dns)
{
    disk_stats_cmd = 'ssh -o StrictHostKeyChecking=no -A jenkins@' + slave_dns + ' "df -kh --output=pcent / | sed -ne 2p | cut -d\"%\" -f1"'
    return sh (returnStdout: true, script: disk_stats_cmd)
}

pipeline
{
    agent none
    environment
    {
        DRY_RUN = "${(params.DRY_RUN != null) ? params.DRY_RUN : true}" //Perform DRY_RUN if no parameter is passed
        SLEEP_INTERVAL = "${(params.SLEEP_INTERVAL != null) ? params.SLEEP_INTERVAL : 2}".toInteger() //Sleep time(in mins) to wait when when a Slave has Active Builds running
        TOT_WAIT_TIME_FOR_SLAVE = "${(params.TOT_WAIT_TIME_FOR_SLAVE != null) ? params.TOT_WAIT_TIME_FOR_SLAVE : 10}".toInteger() //Total time(in mins) to wait if a Slave has Active/Running Builds
        CLEANUP_SLAVES_LABEL= "${(params.CLEANUP_SLAVES_LABEL != null) ? params.CLEANUP_SLAVES_LABEL : "cleanup"}" //Slave
        REPORT_EMAIL = 'CHANGEME@gmail.com'
        JENKINS_URL="https://myjenkins.com"
    }

    stages
    {
        stage('Slaves Disk Cleanup')
        {
            agent { label master}
            steps
            {
                script
                {
                    getNodeNamesWithZeroBuilds(env.CLEANUP_SLAVES_LABEL)
                    getNodeNamesWithRunningBuilds(env.CLEANUP_SLAVES_LABEL)
                    println "Idle Slaves list: " + idleSlaves
                    println "Busy Slaves list: " + busySlaves
                    def allSlavesInSequence = idleSlaves + busySlaves
                    println "All Slaves list in processing sequence: " + allSlavesInSequence
                    println "DRY_RUN value is: " + env.DRY_RUN


                    for (slave_name in allSlavesInSequence)
                    {
                        def waitCounter = 0 //Counter Variable to skip to next Slave if a Slave has active builds running for too long
                        def curl_cmd, slave_dns, workspace_path, disk_used
                        
                        //Change "karthik_jenkins_api_token" to your Jenkins API token
                        withCredentials([string(credentialsId: 'karthik_jenkins_api_token', variable: 'MY_API_TOKEN')])
                        {
                            curl_cmd = "set +x; curl -s -D -u $USERNAME:$MY_API_TOKEN $JENKINS_URL/computer/$slave_name/config.xml | awk -F'<host>|</host>' {'print \$2'}"
                            slave_dns = sh (returnStdout: true, script: curl_cmd).trim ()
                        }

                        println "\n\n=========================================================================="
                        println "Processing ${slave_name}(${slave_dns}) - Active builds: " + getNumberOfRunningBuilds(slave_name)
                        println "=========================================================================="
                        println "Marking ${slave_name} offline for disk cleanup and waiting for pending jobs to complete.."

                        if(passwordlessSSHTest(slave_dns) == 0) 
                        {
                            println(!env.DRY_RUN.toBoolean() ? markSlaveOffline(slave_name) : "DRY_RUN - Consider ${slave_name} marked OFFLINE")
                        } 
                        else 
                        {
                            println "${slave_name} is NOT marked OFFLINE as Passwordless(Public Key based) SSH connectivity seems to be failing"
                            skipSlaves.add(slave_dns)
                            continue
                        }
                        
                        workspace_path = getWorkspaceLocation(slave_name)
                        println "Workspace path of ${slave_dns}: " + workspace_path
                        
                        
                        while(1)
                        {
                            if(getNumberOfRunningBuilds(slave_name)==0)
                            {
                                println "Performing SSH to " + slave_dns

                                if(env.DRY_RUN.toBoolean())
                                {
                                    println "DRY_RUN - hence retrieving disk utilisation statistics"
                                    executeCmdOnSlave(slave_dns,"hostname -f;df -kh /")

                                }
                                else if (isOffline(slave_name)==true && getNumberOfRunningBuilds(slave_name)==0)
                                {
                                    //ansiColor('xterm') 
                                    //{
                                    println "Disk Used % (before Cleanup): " + getDiskUsedPercentage(slave_dns)
                                    println "${slave_dns} - Cleanup of ${workspace_path}/workspace/ In Progress. Please wait.."
                                    executeCmdOnSlave(slave_dns,"docker run -u root \
                                                                -v ${workspace_path}:/workspace \
                                                                alpine:latest \
                                                                /bin/bash -c 'cd /workspace; rm -rf ./-*'")
                                    executeCmdOnSlave(slave_dns,"docker run -u root \
                                                                -v ${workspace_path}:/workspace \
                                                                alpine:latest \
                                                                /bin/bash -c 'cd /workspace; rm -rf *'")

                                    println "${slave_dns} - Cleanup of ${workspace_path}/us-ui-mall/ws/ In Progress. Please wait.."
                                    executeCmdOnSlave(slave_dns,"docker run -u root \
                                                                -v ${workspace_path}:/workspace \
                                                                alpine:latest \
                                                                /bin/bash -c 'cd /workspace; rm -rf *'")

                                    println "${slave_dns} - Cleanup of ${workspace_path}/tw-ui-mall/ In Progress. Please wait.."
                                    executeCmdOnSlave(slave_dns,"docker run -u root \
                                                                -v ${workspace_path}:/workspace \
                                                                alpine:latest \
                                                                /bin/bash -c 'cd /workspace; rm -rf *'")

                                    println "${slave_dns} - Cleanup of ${workspace_path}/../workspace/ In Progress. Please wait.."
                                    executeCmdOnSlave(slave_dns,"docker run -u root \
                                                                -v ${workspace_path}:/workspace \
                                                                alpine:latest \
                                                                /bin/bash -c 'cd /workspace; rm -rf *'")
        
                                    println "${slave_dns} - Cleanup of unused Docker Images, Containers and Files In Progress. Please wait.."
                                    executeCmdOnSlave(slave_dns,"docker system prune -f -a")

                                    println "Disk Used % (after Cleanup): " + getDiskUsedPercentage(slave_dns)
                                    //}
                                }
                                
                                println(!env.DRY_RUN.toBoolean() ? bringSlaveOnline(slave_name) : "DRY_RUN - Consider ${slave_name} brought back ONLINE")
                                doneSlaves.add(slave_dns)
                                break
                            }
                            else 
                            {
                                if (waitCounter >= env.TOT_WAIT_TIME_FOR_SLAVE.toInteger())
                                {
                                  println "Waited for ${waitCounter} minutes. ${slave_name} still has some Active Builds running."
                                  println("Hence, Skipping Disk Cleanup for ${slave_name} and Bringing Slave back online")
                                  bringSlaveOnline(slave_name)
                                  skipSlaves.add(slave_dns)
                                  println "Disk Used % : " + getDiskUsedPercentage(slave_dns)
                                  break
                                }
                                now = new Date(); 
                                println now.format("HH:mm:ss", TimeZone.getTimeZone('IST')) + " IST - Sleeping for ${env.SLEEP_INTERVAL} minutes(for completion of Active/Running builds on that Slave).."

                                sleep(env.SLEEP_INTERVAL.toInteger()*60*1000) {
                                  continue
                                }
                                now = new Date(); 
                                println now.format("HH:mm:ss", TimeZone.getTimeZone('IST')) + " IST - Number of active builds on ${slave_name} : " + getNumberOfRunningBuilds(slave_name)
                                waitCounter += env.SLEEP_INTERVAL.toInteger()
                            }
                        }//while loop
                    }//for loop
                }//Script
            }//Steps
        }//Stage 'Slaves Disk Cleanup'

        stage('Status Report')
        {
            agent any
            steps
            {
                println "Cleanup Finished on : " + doneSlaves
                println "Cleanup Skipped on : " + skipSlaves
            }
        }//Stage 'Status Report'
    }//Stages
    post
    {
        always
        {
            emailext (
            mimeType: "text/html",
            to: "$REPORT_EMAIL",
            subject: "Jenkins Slaves Disk Cleanup '[Build: ${env.BUILD_NUMBER}]' - ${currentBuild.currentResult}",
            body: """
                    <p>Check console output at &QUOT;<a href='${BUILD_URL}consoleFull'>${JOB_NAME} [${BUILD_NUMBER}]</a>&QUOT;</p>
                    <p>Console output:<hr><pre>\${BUILD_LOG_REGEX, regex="^Processing|^===|^Disk|^Cleanup|^Waited|^Hence", linesBefore=0, linesAfter=0, maxMatches=10000, showTruncatedLines=false, escapeHtml=false}</pre></p>"""
            )
        }
        aborted
        {
            println "Aborted"
            //Some Code logic here
        }
    }
}

