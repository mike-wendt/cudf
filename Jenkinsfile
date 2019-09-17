/* Jarvis Helper Functions */
def ghStatusStart(context, description, targetUrl){
    def (account, repo) = env.ghprbGhRepository.tokenize('/')
    githubNotify account: account, context: context, credentialsId: 'gputester-username-api-token', description: description, gitApiUrl: '', repo: repo, sha: env.ghprbActualCommit, status: 'PENDING', targetUrl: targetUrl
}
def ghStatusUpdate(context, description, status, targetUrl){
    if (status == 0) {
        status = 'SUCCESS'
        description += ' - Passed'
    } else {
        status = 'FAILURE'
        description += ' - Failed'
    }
    def (account, repo) = env.ghprbGhRepository.tokenize('/')
    githubNotify account: account, context: context, credentialsId: 'gputester-username-api-token', description: description, gitApiUrl: '', repo: repo, sha: env.ghprbActualCommit, status: status, targetUrl: targetUrl
}

/* Pipeline definition */
node {
    parallel style: {
        node('cpu') {
            stage('Style Check') {
                nvidiaDockerSlavesNode(image: 'gpuci/rapidsai-base:cuda9.2-ubuntu16.04-gcc5-py3.5') {
                    ghStatusStart('jarvis/style','Style Check','')
                    checkout scm
                    def styleCheck = """#!/bin/bash
                    source activate gdf
                    flake8
                    """
                    def statusCode = sh script:styleCheck, returnStatus:true
                    ghStatusUpdate('jarvis/style','Style Check',statusCode,'')
                }
            }
        }
    }, py35_cuda92_ubuntu1604: {
        node('cpu') {
            stage('Conda Build - Py 3.5 CUDA 9.2 Ubuntu 16.04') {
                nvidiaDockerSlavesNode(image: 'gpuci/rapidsai-base:cuda9.2-ubuntu16.04-gcc5-py3.5') {
                    ghStatusStart('jarvis/cudf-cpu-build/py35-cuda92','Conda Build','')
                    checkout scm
                    def buildScript = """#!/bin/bash
set -e
export GIT_DESCRIBE_TAG=`git describe --abbrev=0 --tags`
export GIT_DESCRIBE_NUMBER=`git rev-list \${GIT_DESCRIBE_TAG}..HEAD --count`
export TRAVIS_BRANCH=\${GIT_BRANCH}
export TRAVIS_PULL_REQUEST=\${ghprbPullId}

function logger() {
  echo -e "\n>>>> \$@\n"
}

## Alias to remove travis_retry
function travis_retry() {
  exec \$@
}

## Enable verbose commands
set -x

logger "Get env..."
env

logger "Activate conda env..."
source activate gdf

logger "Install nvStrings..."
conda install -c rapidsai/label/cuda9.2 nvstrings -y

logger "Install cython and gtest..."
conda install -c conda-forge cython=0.28.* gtest=1.8.0 conda-build anaconda-client flake8 conda-verify cmake=3.12 -y

logger "Check versions..."
python --version
gcc --version
g++ --version
conda list

logger "build libcudf..."
source ./travisci/libcudf/build_libcudf.sh
logger "libcudf_cffi..."
source ./travisci/libcudf/build_libcudf_cffi.sh
logger "upload libcudf and libcudf_cffi..."
source ./travisci/libcudf/upload.sh
logger "build cudf..."
source ./travisci/cudf/build_cudf.sh
logger "upload cudf..."
source ./travisci/cudf/upload-anaconda.sh
                    """
                    withEnv(['CUDA=9.2', 'PYTHON=3.5', 'LABEL_MAIN=1', 'BUILD_CUDF=1', 'BUILD_LIBCUDF=1', 'BUILD_CFFI=1']) {
                        def statusCode = sh script:buildScript, returnStatus:true
                        ghStatusUpdate('jarvis/cudf-cpu-conda/py35-cuda92','Conda Build',statusCode,'')
                    }
                }
            }
        }
    },
    failFast: false
}
