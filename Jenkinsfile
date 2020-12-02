#!/usr/bin/groovy

// load pipeline functions
// Requires pipeline-github-lib plugin to load library from github

@Library('github.com/jldeen/jenkins-pipeline@helm3')

def pipeline = new io.estrado.Pipeline()

podTemplate(label: 'jenkins-pipeline', containers: [
    containerTemplate(name: 'jnlp', image: 'jenkinsci/jnlp-slave:3.35-2-alpine', args: '${computer.jnlpmac} ${computer.name}', workingDir: '/home/jenkins', resourceRequestCpu: '200m', resourceLimitCpu: '300m', resourceRequestMemory: '256Mi', resourceLimitMemory: '512Mi'),
    containerTemplate(name: 'docker', image: 'docker:latest', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'helm', image: 'lachlanevenson/k8s-helm:v3.3.4', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl:v1.19.3', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'azcli', image: 'microsoft/azure-cli:latest', command: 'cat', ttyEnabled: true)
],
volumes:[
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
    hostPathVolume(mountPath: '/tmp', hostPath: '/tmp')
],
){

  node ('jenkins-pipeline') {

    def pwd = pwd()
    def chart_dir = "${pwd}/Bikes/charts/bikes"

    checkout scm

    // read in required jenkins workflow config values
    def inputFile = readFile('Jenkinsfile.json')
    def config = new groovy.json.JsonSlurperClassic().parseText(inputFile)
    println "pipeline config ==> ${config}"

    // continue only if pipeline enabled
    if (!config.pipeline.enabled) {
        println "pipeline disabled"
        return
    }

    // set additional git envvars for image tagging
    pipeline.gitEnvVars()

    // If pipeline debugging enabled
    if (config.pipeline.debug) {
      println "DEBUG ENABLED"
      sh "env | sort"

      println "Runing kubectl/helm tests"
      container('kubectl') {
        pipeline.kubectlTest()
      }
      container('helm') {
        pipeline.helmConfig()
      }
    }

    def acct = pipeline.getContainerRepoAcct(config)

    // tag image with version, and branch-commit_id
    def image_tags_map = pipeline.getContainerTags(config)

    // compile tag list
    def image_tags_list = pipeline.getMapValues(image_tags_map)

    stage ('helm lint') {

      container('helm') {

        // run helm chart linter
        pipeline.helmLint(chart_dir)

      }
    }

    stage ('helm package') {

      container('helm') {

        // run helm chart package
        pipeline.helmPackage(chart_dir)
      }
    }
  
  stage ('docker build') {

      container('docker') {

        // perform docker login to container registry as the docker-pipeline-plugin doesn't work with the next auth json format
        withCredentials([[$class          : 'UsernamePasswordMultiBinding', credentialsId: config.container_repo.jenkins_creds_id,
                        usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
          sh "echo ${env.PASSWORD} | docker login -u ${env.USERNAME} --password-stdin ${config.container_repo.host}"
        }

        // dockerbuild
        pipeline.containerBuild(
            dockerfile: config.container_repo.dockerfile,
            host      : config.container_repo.host,
            acct      : acct,
            repo      : config.container_repo.repo,
            tags      : image_tags_list,
            buildTag  : image_tags_list.get(0),
            auth_id   : config.container_repo.jenkins_creds_id
        )
      }
  }
        
  stage ('publish container') {

      container('docker') {

        // perform docker login to container registry as the docker-pipeline-plugin doesn't work with the next auth json format
        withCredentials([[$class          : 'UsernamePasswordMultiBinding', credentialsId: config.container_repo.jenkins_creds_id,
                        usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
          sh "echo ${env.PASSWORD} | docker login -u ${env.USERNAME} --password-stdin ${config.container_repo.host}"
        }

        // publish container
        pipeline.containerPublish(
            dockerfile: config.container_repo.dockerfile,
            host      : config.container_repo.host,
            acct      : acct,
            repo      : config.container_repo.repo,
            tags      : image_tags_list,
            auth_id   : config.container_repo.jenkins_creds_id
        )
      }
  }
    // deploy only the master branch
    if (env.BRANCH_NAME == 'main') {
      stage ('deploy to k8s') {
          // Deploy using Helm chart
        container('helm') {
                    // Create secret from Jenkins credentials manager
          withCredentials([[$class          : 'UsernamePasswordMultiBinding', credentialsId: config.container_repo.jenkins_creds_id,
                        usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
          pipeline.helmDeploy(
            dry_run       : false,
            name          : config.app.name,
            namespace     : config.app.namespace,
            chart_dir     : chart_dir,
            set           : [
              "image.tag": image_tags_list.get(0),
              "image.repository": config.container_repo.image_repository,
              "fullnameOverride": config.app.branch_name,
            ]
          )
          
            //  Run helm tests
            if (config.app.test) {
              pipeline.helmTest(
                name          : config.app.name
              )
            }

            // Kubectl labels
            container('kubectl') {
                sh "kubectl label pods --selector='app=bikes,release=${config.app.name}' routing.visualstudio.io/route-from=bikes -n ${config.app.namespace} --overwrite=true"
          
                sh "kubectl annotate pods --selector='app=bikes,release=${config.app.name}' routing.visualstudio.io/route-on-header=kubernetes-route-as=config.app.branch_name -n ${config.app.namespace} --overwrite=true"
            }
          }
        }
      }

      // GitHub comment
    //   stage ('GitHub PR Comment')
    //     container('')
    // }
  }
}