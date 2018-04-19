pipeline {
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
        sh '''printenv

CWD=`pwd`
PACKAGE=pam_mysql
DOCKER_IMAGE=docker-registry.jc21.net.au/rpmbuild:el7
BUILD_SPEC_ARGS=

mkdir RPMS && chmod -R 777 RPMS

CMD="docker run --rm \\
  --name rpmbuild-$BUILD_TAG \\
  -v $CWD/RPMS:/home/rpmbuilder/rpmbuild/RPMS \\
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
'''
      }
    }
    stage('Publish') {
      steps {
        dir(path: 'RPMS') {
          archiveArtifacts(artifacts: '**/*/*.rpm', caseSensitive: true, onlyIfSuccessful: true)
        }

      }
    }
  }
  triggers {
    bitbucketPush()
  }
}
