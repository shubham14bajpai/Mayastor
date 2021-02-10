#!/usr/bin/env groovy

// On-demand E2E infra configuration
// https://mayadata.atlassian.net/wiki/spaces/MS/pages/247332965/Test+infrastructure#On-Demand-E2E-K8S-Clusters

def e2e_build_cluster_job='k8s-build-cluster' // Jenkins job to build cluster
def e2e_destroy_cluster_job='k8s-destroy-cluster' // Jenkins job to destroy cluster
// Environment to run e2e test in (job param of $e2e_build_cluster_job)
def e2e_environment="hcloud-kubeadm"
// Global variable to pass current k8s job between stages
def k8s_job=""
def image_tag='v0.7.0'

// Searches previous builds to find first non aborted one
def getLastNonAbortedBuild(build) {
  if (build == null) {
    return null;
  }

  if(build.result.toString().equals("ABORTED")) {
    return getLastNonAbortedBuild(build.getPreviousBuild());
  } else {
    return build;
  }
}

// Send out a slack message if branch got broken or has recovered
def notifySlackUponStateChange(build) {
  def cur = build.getResult()
  def prev = getLastNonAbortedBuild(build.getPreviousBuild())?.getResult()
  if (cur != prev) {
    if (cur == 'SUCCESS') {
      slackSend(
        channel: '#mayastor-backend',
        color: 'normal',
        message: "Branch ${env.BRANCH_NAME} has been fixed :beers: (<${env.BUILD_URL}|Open>)"
      )
    } else if (prev == 'SUCCESS') {
      slackSend(
        channel: '#mayastor-backend',
        color: 'danger',
        message: "Branch ${env.BRANCH_NAME} is broken :face_with_raised_eyebrow: (<${env.BUILD_URL}|Open>)"
      )
    }
  }
}

// Will ABORT current job for cases when we don't want to build
if (currentBuild.getBuildCauses('jenkins.branch.BranchIndexingCause') &&
    BRANCH_NAME == "develop") {
    print "INFO: Branch Indexing, aborting job."
    currentBuild.result = 'ABORTED'
    return
}

// Only schedule regular builds on develop branch, so we don't need to guard against it
String cron_schedule = BRANCH_NAME == "develop" ? "0 2 * * *" : ""

// Determine which stages to run
if (param.e2e_continuous == true) {
  run_linter = false
  rust_test = false
  grpc_test = false
  moac_test = false
  e2e_test = true
  e2e_image_build = false
  allow_push_images = false
} else {
  run_linter = true
  rust_test = true
  grpc_test = true
  moac_test = true
  e2e_test = true
  e2e_image_build = true
  allow_push_images = true
}

pipeline {
  agent none
  options {
    timeout(time: 2, unit: 'HOURS')
  }
  parameters {
    booleanParam(defaultValue: false, name: 'e2e_continuous')
  }
  triggers {
    cron(cron_schedule)
  }

  stages {
    stage('init') {
      agent { label 'nixos-mayastor' }
      steps {
        step([
          $class: 'GitHubSetCommitStatusBuilder',
          contextSource: [
            $class: 'ManuallyEnteredCommitContextSource',
            context: 'continuous-integration/jenkins/branch'
          ],
          statusMessage: [ content: 'Pipeline started' ]
        ])
      }
    }
    stage('linter') {
      agent { label 'nixos-mayastor' }
      when {
        beforeAgent true
        not {
          anyOf {
            branch 'master'
            branch 'release/*'
            expression { run_linter == false }
          }
        }
      }
      steps {
        sh 'nix-shell --run "cargo fmt --all -- --check"'
        sh 'nix-shell --run "cargo clippy --all-targets -- -D warnings"'
        sh 'nix-shell --run "./scripts/js-check.sh"'
      }
    }
    stage('test') {
      when {
        beforeAgent true
        not {
          anyOf {
            branch 'master'
            branch 'release/*'
          }
        }
      }
      parallel {
        stage('rust unit tests') {
          when{
            expression { rust_test == true }
          }
          agent { label 'nixos-mayastor' }
          steps {
            sh 'printenv'
            sh 'nix-shell --run "./scripts/cargo-test.sh"'
          }
          post {
            always {
              // in case of abnormal termination of any nvmf test
              sh 'sudo nvme disconnect-all'
            }
          }
        }
        stage('grpc tests') {
          when{
            expression { grpc_test == true }
          }
          agent { label 'nixos-mayastor' }
          steps {
            sh 'printenv'
            sh 'nix-shell --run "./scripts/grpc-test.sh"'
          }
          post {
            always {
              junit '*-xunit-report.xml'
            }
          }
        }
        stage('moac unit tests') {
          when{
            expression { moac_test == true }
          }
          agent { label 'nixos-mayastor' }
          steps {
            sh 'printenv'
            sh 'nix-shell --run "./scripts/moac-test.sh"'
          }
          post {
            always {
              junit 'moac-xunit-report.xml'
            }
          }
        }
        stage('e2e tests') {
          when{
            expression { e2e_test == true }
          }
          stages {
            stage('e2e docker images') {
              when{
                expression { e2e_image_build == true }
              }
              agent { label 'nixos-mayastor' }
              steps {
                // e2e tests are the most demanding step for space on the disk so we
                // test the free space here rather than repeating the same code in all
                // stages.
                sh "./scripts/reclaim-space.sh 10"
                // Build images (REGISTRY is set in jenkin's global configuration).
                // Note: We might want to build and test dev images that have more
                // assertions instead but that complicates e2e tests a bit.
                sh "./scripts/release.sh --alias-tag ci --registry \"${env.REGISTRY}\""
                // Always remove all docker images because they are usually used just once
                // and underlaying pkgs are already cached by nix so they can be easily
                // recreated.
              }
              post {
                always {
                  sh 'docker image prune --all --force'
                }
              }
            }
            stage('build e2e cluster') {
              agent { label 'nixos' }
              steps {
                script {
                  k8s_job=build(
                    job: "${e2e_build_cluster_job}",
                    propagate: true,
                    wait: true,
                    parameters: [[
                      $class: 'StringParameterValue',
                      name: "ENVIRONMENT",
                      value: "${e2e_environment}"
                    ]]
                  )
                }
              }
            }
            stage('run e2e') {
              agent { label 'nixos-mayastor' }
              environment {
                KUBECONFIG = "${env.WORKSPACE}/${e2e_environment}/modules/k8s/secrets/admin.conf"
              }
              steps {
                script (
                  // FIXME(arne-rusek): move hcloud's config to top-level dir in TF scripts

                  sh """
                    mkdir -p "${e2e_environment}/modules/k8s/secrets"
                  """
                  copyArtifacts(
                      projectName: "${k8s_job.getProjectName()}",
                      selector: specific("${k8s_job.getNumber()}"),
                      filter: "${e2e_environment}/modules/k8s/secrets/admin.conf",
                      target: "",
                      fingerprintArtifacts: true
                  )
                  sh 'kubectl get nodes -o wide'

                  def tag = ""
                  if (e2e_image_build == false) {
                    tag = image_tag
                  } else {
                    tag = sh(
                      // using printf to get rid of trailing newline
                      script: "printf \$(git rev-parse --short ${GIT_COMMIT})",
                      returnStdout: true
                    )
                  }
                  sh "nix-shell --run './scripts/e2e-test.sh --device /dev/sdb --tag \"${tag}\" --registry \"${env.REGISTRY}\"'"
                }
              }
            }
            stage('destroy e2e cluster') {
              agent { label 'nixos' }
              steps {
                script {
                  build(
                    job: "${e2e_destroy_cluster_job}",
                    propagate: true,
                    wait: true,
                    parameters: [
                      [
                        $class: 'StringParameterValue',
                        name: "ENVIRONMENT",
                        value: "${e2e_environment}"
                      ],
                      [
                        $class: 'RunParameterValue',
                        name: "BUILD",
                        runId:"${k8s_job.getProjectName()}#${k8s_job.getNumber()}"
                      ]
                    ]
                  )
                }
              }
            }
          }
        }
      }
    }
    stage('push images') {
      agent { label 'nixos-mayastor' }
      when {
        beforeAgent true
        allOf {
          expression { allow_push_images == true }
          anyOf {
            branch 'master'
            branch 'release/*'
            branch 'develop'
          }
        }
      }
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
          sh 'echo $PASSWORD | docker login -u $USERNAME --password-stdin'
        }
        sh './scripts/release.sh'
      }
      post {
        always {
          sh 'docker logout'
          sh 'docker image prune --all --force'
        }
      }
    }
  }

  // The main motivation for post block is that if all stages were skipped
  // (which happens when running cron job and branch != develop) then we don't
  // want to set commit status in github (jenkins will implicitly set it to
  // success).
  post {
    always {
      node(null) {
        script {
          // If no tests were run then we should neither be updating commit
          // status in github nor send any slack messages
          if (currentBuild.result != null) {
            step([
              $class: 'GitHubCommitStatusSetter',
              errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
              contextSource: [
                $class: 'ManuallyEnteredCommitContextSource',
                context: 'continuous-integration/jenkins/branch'
              ],
              statusResultSource: [
                $class: 'ConditionalStatusResultSource',
                results: [
                  [$class: 'AnyBuildResult', message: 'Pipeline result', state: currentBuild.getResult()]
                ]
              ]
            ])
            if (env.BRANCH_NAME == 'develop') {
              notifySlackUponStateChange(currentBuild)
            }
          }
        }
      }
    }
  }
}
