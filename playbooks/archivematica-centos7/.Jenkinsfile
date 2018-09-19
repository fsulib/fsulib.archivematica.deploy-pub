node {
timestamps{
stage('Get code') {
   // If environment variables are defined, honour them
   env.AM_BRANCH = sh(script: 'echo ${AM_BRANCH:-"stable/1.7.x"}', returnStdout: true).trim()
   env.SS_BRANCH = sh(script: 'echo ${SS_BRANCH:-"stable/0.12.x"}', returnStdout: true).trim()
   env.DISPLAY = sh(script: 'echo ${DISPLAY:-:60}', returnStdout: true).trim()
   env.ACCEPTANCE_TAGS = sh(script: 'echo ${ACCEPTANCE_TAGS:-"uuids-dirs mo-aip-reingest ipc icc tpc picc aip-encrypt"}', returnStdout: true).trim()
   env.VAGRANT_VAGRANTFILE = sh(script: 'echo ${VAGRANT_VAGRANTFILE:-Vagrantfile.openstack}', returnStdout: true).trim()
   env.OS_IMAGE = sh(script: 'echo ${OS_IMAGE:-"Centos 7"}', returnStdout: true).trim()

   // Set build name
   currentBuild.displayName = "AM:${AM_BRANCH} SS:${SS_BRANCH}."
   currentBuild.description = "OS: Centos 7 <br>Tests: ${ACCEPTANCE_TAGS}"

   git branch: env.AM_BRANCH, poll: false,
       url: 'https://github.com/artefactual/archivematica'
   git branch: env.SS_BRANCH, poll: false,
       url: 'https://github.com/artefactual/archivematica-storage-service'

   checkout([$class: 'GitSCM',
            branches: [[name: 'dev/add-jenkins-centos']],
            doGenerateSubmoduleConfigurations: false,
            extensions:
            [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'deploy-pub']],
            submoduleCfg: [],
            userRemoteConfigs: [[url: 'https://github.com/artefactual/deploy-pub']]])

}

stage ('Create vm') {
sh '''
   echo Building Archivematica $AM_BRANCH and Storage Service $SS_BRANCH
   cd deploy-pub/playbooks/archivematica-centos7
   source ~/.secrets/openrc.sh
   ansible-galaxy install -f -p roles -r requirements.yml
   export ANSIBLE_ARGS="-e archivematica_src_am_version=${AM_BRANCH} archivematica_src_ss_version=${SS_BRANCH}"
   vagrant up
   vagrant ssh-config | tee >( grep HostName  | awk '{print $2}' > $WORKSPACE/.host) \
                            >( grep User | awk '{print $2}' > $WORKSPACE/.user ) \
                            >( grep IdentityFile | awk '{print $2}' > $WORKSPACE/.key )

   vagrant ssh -c "sudo chmod 755 /home/centos"
   vagrant ssh -c "sudo usermod -aG archivematica centos"
   vagrant ssh -c "sudo service firewalld stop"
'''
env.SERVER = sh(script: "cat .host", returnStdout: true).trim()
env.USER = sh(script: "cat .user", returnStdout: true).trim()
env.KEY = sh(script: "cat .key", returnStdout: true).trim()
}

stage('Configure acceptance tests') {
git branch: 'master', url: 'https://github.com/artefactual-labs/archivematica-acceptance-tests'
   properties([disableConcurrentBuilds(),
    gitLabConnection(''),
    [$class: 'RebuildSettings',
    autoRebuild: false,
    rebuildDisabled: false],
    pipelineTriggers([pollSCM('*/5 * * * *')])])


sh '''
    virtualenv -p python3 env
    env/bin/pip install -r requirements.txt
    env/bin/pip install behave2cucumber
    # Launch vnc server
    VNCPID=$(ps aux | grep Xtig[h] | grep ${DISPLAY} | awk '{print $2}')
    if [ "x$VNCPID" == "x" ]; then
       tightvncserver -geometry 1920x1080 ${DISPLAY}
    fi

    mkdir -p results/
    rm -rf results/*
'''
}

stage('Run tests') {
sh '''
for i in $ACCEPTANCE_TAGS
do
    timeout 30m env/bin/behave \
          --tags=$i \
          --no-skipped \
          -D am_version=1.7 \
          -D driver_name=Firefox \
          -D am_username=admin \
          -D am_password=archivematica \
          -D am_url=http://${SERVER}/ \
          -D ss_username=admin \
          -D ss_password=archivematica \
          -D ss_url=http://${SERVER}:8000/ \
          -D home=${USER} \
          -D server_user=${USER} \
          -D transfer_source_path=${USER}/archivematica-sampledata/TestTransfers/acceptance-tests \
          -D ssh_identity_file=${KEY} \
           --junit --junit-directory=results/ -v \
            -f=json -o=results/output-$i.json \
            --no-skipped || true

      env/bin/python -m behave2cucumber  -i results/output-$i.json -o results/cucumber-$i.json || true
done
'''
}
stage('Archive results'){
junit allowEmptyResults: true, keepLongStdio: true, testResults: 'results/*.xml'
cucumber 'results/cucumber-*.json'
}
stage('Cleanup') {
sh '''
    # Kill vnc server
    VNCPID=$(ps aux | grep Xtig[h] | grep ${DISPLAY} | awk '{print $2}')
    if [ "x$VNCPID" != "x" ]; then
       kill $VNCPID
    fi
    # Remove vm
    cd deploy-pub/playbooks/archivematica-centos7/
    source ~/.secrets/openrc.sh
    vagrant destroy
'''
}
}
}