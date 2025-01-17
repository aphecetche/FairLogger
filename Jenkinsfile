#!groovy

def jobMatrix(String prefix, String type, List specs) {
  def nodes = [:]
  for (spec in specs) {
    job = "${spec.os}-${spec.ver}-${spec.arch}-${spec.compiler}"
    def label = "${type}/${job}"
    def selector = "${spec.os}-${spec.ver}-${spec.arch}"
    def os = spec.os
    def ver = spec.ver
    def check = spec.check

    nodes[label] = {
      node(selector) {
        githubNotify(context: "${prefix}/${label}", description: 'Building ...', status: 'PENDING')
        try {
          deleteDir()
          checkout scm

          def jobscript = 'job.sh'
          def ctestcmd = "ctest -S FairLoggerTest.cmake -V --output-on-failure"
          sh "echo \"set -e\" >> ${jobscript}"
          sh "echo \"export LABEL=\\\"\${JOB_BASE_NAME} ${label}\\\"\" >> ${jobscript}"
          if (selector =~ /^macos/) {
            sh "echo \"${ctestcmd}\" >> ${jobscript}"
            sh "cat ${jobscript}"
            sh "bash ${jobscript}"
          } else {
            def containercmd = "singularity exec -B/shared ${env.SINGULARITY_CONTAINER_ROOT}/fairlogger/${os}.${ver}.sif bash -l -c \\\"${ctestcmd}\\\""
            sh """\
              echo \"echo \\\"*** Job started at .......: \\\$(date -R)\\\"\" >> ${jobscript}
              echo \"echo \\\"*** Job ID ...............: \\\${SLURM_JOB_ID}\\\"\" >> ${jobscript}
              echo \"echo \\\"*** Compute node .........: \\\$(hostname -f)\\\"\" >> ${jobscript}
              echo \"unset http_proxy\" >> ${jobscript}
              echo \"unset HTTP_PROXY\" >> ${jobscript}
              echo \"${containercmd}\" >> ${jobscript}
            """
            sh "cat ${jobscript}"
            sh "test/ci/slurm-submit.sh \"FairLogger \${JOB_BASE_NAME} ${label}\" ${jobscript}"
          }

          deleteDir()
          githubNotify(context: "${prefix}/${label}", description: 'Success', status: 'SUCCESS')
        } catch (e) {
          deleteDir()
          githubNotify(context: "${prefix}/${label}", description: 'Error', status: 'ERROR')
          throw e
        }
      }
    }
  }
  return nodes
}

pipeline{
  agent none
  stages {
    stage("Run CI Matrix") {
      steps{
        script {
          def builds = jobMatrix('alfa-ci', 'build', [
            [os: 'fedora', ver: '32',    arch: 'x86_64', compiler: 'gcc-10'],
            [os: 'fedora', ver: '33',    arch: 'x86_64', compiler: 'gcc-10'],
            [os: 'fedora', ver: '34',    arch: 'x86_64', compiler: 'gcc-11'],
            [os: 'macos',  ver: '11',    arch: 'x86_64', compiler: 'apple-clang-12'],
            [os: 'macos',  ver: '11',    arch: 'arm64',  compiler: 'apple-clang-13'],
          ])

          parallel(builds)
        }
      }
    }
  }
}
