#!/usr/bin/env groovy
// groovylint-disable DuplicateStringLiteral, NestedBlockDepth
// groovylint-disable VariableName
//
// Copyright (c) 2020-2023 Intel Corporation
//
// Permission is hereby granted, free of charge, to any person obtaining a
// copy of this software and associated documentation files (the "Software"),
// to deal in the Software without restriction, including without limitation
// the rights to use, copy, modify, merge, publish, distribute, sublicense,
// and/or sell copies of the Software, and to permit persons to whom the
// Software is furnished to do so, subject to the following conditions:
//
// The above copyright notice and this permission notice shall be included in
// all copies or substantial portions of the Software.
//
// THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
// IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
// FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
// THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
// LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
// FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
// DEALINGS IN THE SOFTWARE.

// To use a test branch (i.e. PR) until it lands to master
// I.e. for testing library changes
//@Library(value='pipeline-lib@your_branch') _

// groovylint-disable-next-line CompileStatic
String sanitized_JOB_NAME = JOB_NAME.toLowerCase().replaceAll(
                                               '/', '-').replaceAll('%2f', '-')

String docker_http_proxy() {
    String proxy = ''
    if (env.HTTP_PROXY) {
        proxy += ' --build-arg HTTP_PROXY="' + env.HTTP_PROXY + '"' +
                 ' --build-arg http_PROXY="' + env.HTTP_PROXY + '"'
    }
    if (env.HTTPS_PROXY) {
        proxy += ' --build-arg HTTPS_PROXY="' + env.HTTPS_PROXY + '"' +
                 ' --build-arg https_PROXY="' + env.HTTPS_PROXY + '"'
    }
    return proxy
}

pipeline {
    agent { label 'lightweight' }

    environment {
        UID = sh(script: 'id -u', returnStdout: true)
        BUILDARGS = "--build-arg UID=${env.UID} ${docker_http_proxy()}"
        // GIT_CHECKOUT_DIR not set by github multibranch pipeline
        GIT_CHECKOUT_DIR = 'repo'
    }

    options {
        checkoutToSubdirectory('repo')
        ansiColor('xterm')
    }

    stages {
        stage('Cancel Previous Builds') {
            when {
                beforeAgent true
                expression { !skipStage() }
            }
            steps {
                cancelPreviousBuilds()
            } // steps
        } // stage('Cancel Previous Builds')

        stage('Malware Scan Lint') {
            agent {
                dockerfile {
                    filename 'Dockerfile.code_scanning'
                    dir env.GIT_CHECKOUT_DIR
                    label 'docker_runner'
                    additionalBuildArgs '$BUILDARGS' +
                                        "-t ${sanitized_JOB_NAME}-fedora "
                } // dockerfile
            } // agent
            steps {
                sh label: 'Build test_env',
                   script: "${env.GIT_CHECKOUT_DIR}/bin/ci-test-venv"
                sh label: 'Malware Scan',
                    script: "${env.GIT_CHECKOUT_DIR}/bin/ci-maldet"
                checkoutScm url: 'https://github.com/daos-stack/code_review.git',
                            checkoutDir: 'code_review'
                sh label: 'Linting',
                   script: '''PROJECT_REPO=$PWD/"$GIT_CHECKOUT_DIR"
                              export PROJECT_REPO
                              code_review/checkpatch_lint.sh'''
            } //steps
            post {
                always {
                    // pipeline-lib wraps archiveArtifacts, and currently
                    // assumes that the SCM is checked out in the base of the
                    // workspace.
                    archiveArtifacts artifacts: '../pylint.log',
                                     allowEmptyArchive: true
                } // always
            } // post
        } // stage ('Malware Scan Lint')
        stage('Test') {
            agent {
                label 'docker_runner'
            } // agent
            steps {
                withCredentials(
                   [usernamePassword(
                     credentialsId: 'daos_1source_project_github_2022',
                     passwordVariable: 'GIT_PASSWORD',
                     usernameVariable: 'GIT_USERNAME')]) {
                    sh label: 'Setup ~/.netrc part1',
                       script: 'printf "machine github.com\n ' +
                               '  login not-used\n' +
                               '  password ' + GIT_PASSWORD + '\n" >> ~/.netrc'
                    sh label: 'Setup ~/.netrc part2',
                       script: 'chmod 0700 ~/.netrc'
                }
                sh label: 'Test role',
                   script: "${env.GIT_CHECKOUT_DIR}/bin/ci-test-role"
            } //steps
        } // stage('Lint')
    } // stages
}
