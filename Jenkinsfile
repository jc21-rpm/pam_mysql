pipeline {
  options {
    buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '10'))
    disableConcurrentBuilds()
  }
  agent any
  stages {
    stage('Prepare') {
      steps {
        sh '''docker pull docker-registry.jc21.net.au/rpmbuild:el7
docker pull docker-registry.jc21.net.au/ci-tools:latest'''
      }
    }
    stage('Build') {
      steps {
        sh '''CWD=`pwd`
PACKAGE=pam_mysql
DOCKER_IMAGE=docker-registry.jc21.net.au/rpmbuild:el7
BUILD_SPEC_ARGS=

mkdir -p RPMS && chmod -R 777 RPMS
mkdir -p SRPMS && chmod -R 777 SRPMS

CMD="docker run --rm \\
  --name rpmbuild-$BUILD_TAG \\
  -v $CWD/RPMS:/home/rpmbuilder/rpmbuild/RPMS \\
  -v $CWD/SRPMS:/home/rpmbuilder/rpmbuild/SRPMS \\
  -v $CWD/SPECS:/home/rpmbuilder/rpmbuild/SPECS \\
  -v $CWD/SOURCES:/home/rpmbuilder/rpmbuild/SOURCES \\
  $DOCKER_IMAGE \\
  /bin/build-spec $BUILD_SPEC_ARGS -- /home/rpmbuilder/rpmbuild/SPECS/$PACKAGE.spec"

$CMD

exit $?'''
      }
    }
    stage('Sign') {
      steps {
        sh '''rm -rf sign
mkdir -p sign
exit 0'''
        dir(path: 'sign') {
          git(url: 'ssh://git@stash.jc21.com:7999/rpm/rpm-sign.git', credentialsId: 'stash.jc21.com')
          sh 'chmod 600 .gnupg/*'
        }

        sh '''CWD=`pwd`
DOCKER_IMAGE=docker-registry.jc21.net.au/ci-tools:latest

for RPMFILE in RPMS/*/*.rpm
do
  CMD="docker run --rm \\
    --name rpmbuild-$BUILD_TAG \\
    -v $CWD/RPMS:/data/RPMS \\
    -v $CWD/sign:/data/sign \\
    -v $CWD/sign/.rpmmacros:/root/.rpmmacros \\
    -v $CWD/sign/.gnupg:/root/.gnupg \\
    $DOCKER_IMAGE \\
    /data/sign/addsign.exp /data/$RPMFILE"

  $CMD

  # exit if bad return code
  rc=$?; if [[ $rc != 0 ]]; then exit $rc; fi
done

# source rpms
for RPMFILE in SRPMS/*.src.rpm
do
  CMD="docker run --rm \\
    --name rpmbuild-$BUILD_TAG \\
    -v $CWD/SRPMS:/data/SRPMS \\
    -v $CWD/sign:/data/sign \\
    -v $CWD/sign/.rpmmacros:/root/.rpmmacros \\
    -v $CWD/sign/.gnupg:/root/.gnupg \\
    $DOCKER_IMAGE \\
    /data/sign/addsign.exp /data/$RPMFILE"

  $CMD

  # exit if bad return code
  rc=$?; if [[ $rc != 0 ]]; then exit $rc; fi
done
'''
      }
    }
    stage('Publish') {
      steps {
        dir(path: 'RPMS') {
          archiveArtifacts(artifacts: '**/*/*.rpm', caseSensitive: true, onlyIfSuccessful: true)
        }

        dir(path: 'SRPMS') {
          archiveArtifacts(artifacts: '**/*.src.rpm', caseSensitive: true, onlyIfSuccessful: true, allowEmptyArchive: true)
        }
      }
    }
  }
  triggers {
    bitbucketPush()
  }
  post {
    success {
      slackSend color: "#72c900", message: "SUCCESS: <${BUILD_URL}|${JOB_NAME}> build #${BUILD_NUMBER}"
    }
    failure {
      slackSend color: "#d61111", message: "FAILED: <${BUILD_URL}|${JOB_NAME}> build #${BUILD_NUMBER}"
    }
  }
}
