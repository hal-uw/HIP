#!/usr/bin/env groovy

// Generated from snippet generator 'properties; set job properties'
properties([buildDiscarder(logRotator(
    artifactDaysToKeepStr: '',
    artifactNumToKeepStr: '',
    daysToKeepStr: '',
    numToKeepStr: '10')),
    disableConcurrentBuilds(),
    parameters([booleanParam( name: 'push_image_to_docker_hub', defaultValue: false, description: 'Push hip & hcc image to rocm docker-hub' )]),
    [$class: 'CopyArtifactPermissionProperty', projectNames: '*']
   ])

////////////////////////////////////////////////////////////////////////
// -- AUXILLARY HELPER FUNCTIONS

////////////////////////////////////////////////////////////////////////
// Return build number of upstream job
@NonCPS
int get_upstream_build_num( )
{
    def upstream_cause = currentBuild.rawBuild.getCause( hudson.model.Cause$UpstreamCause )
    if( upstream_cause == null)
      return 0

    return upstream_cause.getUpstreamBuild()
}

////////////////////////////////////////////////////////////////////////
// Return project name of upstream job
@NonCPS
String get_upstream_build_project( )
{
    def upstream_cause = currentBuild.rawBuild.getCause( hudson.model.Cause$UpstreamCause )
    if( upstream_cause == null)
      return null

    return upstream_cause.getUpstreamProject()
}

////////////////////////////////////////////////////////////////////////
// Construct the relative path of the build directory
String build_directory_rel( String build_config )
{
  if( build_config.equalsIgnoreCase( 'release' ) )
  {
    return "build/release"
  }
  else
  {
    return "build/debug"
  }
}

////////////////////////////////////////////////////////////////////////
// Lots of images are created above; no apparent way to delete images:tags with docker global variable
def docker_clean_images( String org, String image_name )
{
  // Check if any images exist first grepping for image names
  int docker_images = sh( script: "docker images | grep \"${org}/${image_name}\"", returnStatus: true )

  // The script returns a 0 for success (images were found )
  if( docker_images == 0 )
  {
    // Deleting images can fail, if other projects have built on top of that image and are now dependent on it.
    // This should not be treated as a hip build failure.  This requires cleanup at a later time, possibly through
    // another job
    try
    {
      // Best attempt to run bash script to clean images
      // deleting images based on hash seems to be more stable than through name:tag values because of <none> tags
      sh "docker images | grep \"${org}/${image_name}\" | awk '{print \$1 \":\" \$2}' | xargs docker rmi"
    }
    catch( err )
    {
      println 'Failed to cleanup a few images; probably the images are used as a base for other images'
      currentBuild.result = 'SUCCESS'
    }
  }
}

////////////////////////////////////////////////////////////////////////
// -- BUILD RELATED FUNCTIONS

////////////////////////////////////////////////////////////////////////
// Checkout source code, source dependencies and update version number numbers
// Returns a relative path to the directory where the source exists in the workspace
String checkout_and_version( String platform )
{
  String source_dir_rel = "src"
  String source_hip_rel = "${source_dir_rel}/hip"

  stage("${platform} clone")
  {
    dir( "${source_hip_rel}" )
    {
      // checkout hip
      checkout([
        $class: 'GitSCM',
        branches: scm.branches,
        doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
        extensions: scm.extensions + [[$class: 'CleanCheckout']],
        userRemoteConfigs: scm.userRemoteConfigs
      ])
    }
  }

  return source_hip_rel
}


////////////////////////////////////////////////////////////////////////
// This creates the docker image that we use to build the project in
// The docker images contains all dependencies, including OS platform, to build
def docker_build_image( String platform, String org, String optional_build_parm, String source_hip_rel, String from_image )
{
  String build_image_name = "build-ubuntu-16.04"
  String dockerfile_name = "dockerfile-build-ubuntu-16.04"
  def build_image = null

  stage("${platform} build image")
  {
    dir("${source_hip_rel}")
    {
      def user_uid = sh( script: 'id -u', returnStdout: true ).trim()

      // Docker 17.05 introduced the ability to use ARG values in FROM statements
      // Docker inspect failing on FROM statements with ARG https://issues.jenkins-ci.org/browse/JENKINS-44836
      // build_image = docker.build( "${org}/${build_image_name}:latest", "--pull -f docker/${dockerfile_name} --build-arg user_uid=${user_uid} --build-arg base_image=${from_image} ." )

      // JENKINS-44836 workaround by using a bash script instead of docker.build()
      sh "docker build -t ${org}/${build_image_name}:latest -f docker/${dockerfile_name} ${optional_build_parm} --build-arg user_uid=${user_uid} --build-arg base_image=${from_image} ."
      build_image = docker.image( "${org}/${build_image_name}:latest" )
    }
  }

  return build_image
}

////////////////////////////////////////////////////////////////////////
// This encapsulates the cmake configure, build and package commands
// Leverages docker containers to encapsulate the build in a fixed environment
def docker_build_inside_image( def build_image, String inside_args, String platform, String optional_configure, String build_config, String source_hip_rel, String build_dir_rel )
{
  String source_hip_abs = pwd() + "/" + source_hip_rel

  build_image.inside( inside_args )
  {
    stage("${platform} make ${build_config}")
    {
      // The rm command needs to run as sudo because the test steps below create files owned by root
      sh  """#!/usr/bin/env bash
          set -x
          sudo rm -rf ${build_dir_rel}
          mkdir -p ${build_dir_rel}
          cd ${build_dir_rel}
          cmake -DCMAKE_BUILD_TYPE=${build_config} -DCMAKE_INSTALL_PREFIX=staging ${optional_configure} ${source_hip_abs}
          make -j\$(nproc)
        """
    }

    // Cap the maximum amount of testing, in case of hangs
    timeout(time: 1, unit: 'HOURS')
    {
      stage("${platform} unit testing")
      {
        sh  """#!/usr/bin/env bash
            set -x
            cd ${build_dir_rel}
            make install -j\$(nproc)
            make build_tests -i -j\$(nproc)
            make test
          """
        // If unit tests output a junit or xunit file in the future, jenkins can parse that file
        // to display test results on the dashboard
        // junit "${build_dir_rel}/*.xml"
      }
    }

    // Only create packages from hcc based builds
    if( platform.toLowerCase( ).startsWith( 'hcc-' ) )
    {
      stage("${platform} packaging")
      {
        sh  """#!/usr/bin/env bash
            set -x
            cd ${build_dir_rel}
            make package
          """

        // No matter the base platform, all packages have the same name
        // Only upload 1 set of packages, so we don't have a race condition uploading packages
        if( platform.toLowerCase( ).startsWith( 'hcc-ctu' ) )
        {
          archiveArtifacts artifacts: "${build_dir_rel}/*.deb", fingerprint: true
          archiveArtifacts artifacts: "${build_dir_rel}/*.rpm", fingerprint: true
        }
      }
    }
  }

  return void
}

////////////////////////////////////////////////////////////////////////
// This builds a fresh docker image FROM a clean base image, with no build dependencies included
// Uploads the new docker image to internal artifactory
String docker_upload_artifactory( String hcc_ver, String artifactory_org, String from_image, String source_hip_rel, String build_dir_rel )
{
  def hip_install_image = null
  String image_name = "hip-${hcc_ver}-ubuntu-16.04"

  stage( 'artifactory' )
  {
    println "artifactory_org: ${artifactory_org}"

    //  We copy the docker files into the bin directory where the .deb lives so that it's a clean build everytime
    sh "cp -r ${source_hip_rel}/docker/* ${build_dir_rel}"

    // Docker 17.05 introduced the ability to use ARG values in FROM statements
    // Docker inspect failing on FROM statements with ARG https://issues.jenkins-ci.org/browse/JENKINS-44836
    // hip_install_image = docker.build( "${artifactory_org}/${image_name}:${env.BUILD_NUMBER}", "--pull -f ${build_dir_rel}/dockerfile-hip-ubuntu-16.04 --build-arg base_image=${from_image} ${build_dir_rel}" )

    // JENKINS-44836 workaround by using a bash script instead of docker.build()
    sh "docker build -t ${artifactory_org}/${image_name} --pull -f ${build_dir_rel}/dockerfile-hip-ubuntu-16.04 --build-arg base_image=${from_image} ${build_dir_rel}"
    hip_install_image = docker.image( "${artifactory_org}/${image_name}" )

    // The connection to artifactory can fail sometimes, but this should not be treated as a build fail
    try
    {
      // Don't push pull requests to artifactory, these tend to accumulate over time
      if( env.BRANCH_NAME.toLowerCase( ).startsWith( 'pr-' ) )
      {
        println 'Pull Request (PR-xxx) detected; NOT pushing to artifactory'
      }
      else
      {
        docker.withRegistry('http://compute-artifactory:5001', 'artifactory-cred' )
        {
          hip_install_image.push( "${env.BUILD_NUMBER}" )
          hip_install_image.push( 'latest' )
        }
      }
    }
    catch( err )
    {
      currentBuild.result = 'SUCCESS'
    }
  }

  return image_name
}

////////////////////////////////////////////////////////////////////////
// Uploads the new docker image to the public docker-hub
def docker_upload_dockerhub( String local_org, String image_name, String remote_org )
{
  stage( 'docker-hub' )
  {
    // Do not treat failures to push to docker-hub as a build fail
    try
    {
      sh  """#!/usr/bin/env bash
          set -x
          echo inside sh
          docker tag ${local_org}/${image_name} ${remote_org}/${image_name}
        """

      docker_hub_image = docker.image( "${remote_org}/${image_name}" )

      docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-cred' )
      {
        docker_hub_image.push( "${env.BUILD_NUMBER}" )
        docker_hub_image.push( 'latest' )
      }
    }
    catch( err )
    {
      currentBuild.result = 'SUCCESS'
    }
  }
}

////////////////////////////////////////////////////////////////////////
// hcc_integration_testing
// This function is sets up compilation and testing of HiP on a compiler downloaded from an upstream build
// Integration testing is centered around docker and constructing clean test environments every time

// NOTES: I have implemeneted integration testing 3 different ways, and I've come to the conclusion nothing is perfect
// 1.  I've tried having HCC push the test compiler to artifactory, and having HiP download the test docker image from artifactory
//     a.  The act of uploading and downloading images from artifactory takes minutes
//     b.  There is no good way of deleting images from a repository.  You have to use an arcane CURL command and I don't know how
//        to keep the password secret.  These test integration images are meant to be ephemeral.
// 2.  I tried 'docker save' to export a docker image into a tarball, and transfering the image through 'copy artifacts plugin'
//     a.  The HCC docker image uncompressed is over 1GB
//     b.  Compressing the docker image takes even longer than uploading the image to artifactory
// 3.  Download the HCC .deb and dockerfile through 'copy artifacts plugin'.  Create a new HCC image on the fly
//     a.  There is inefficency in building a new ubuntu image and installing HCC twice (once in HCC build, once here)
//     b.  This solution doesn't scale when we start testing downstream libraries

// I've implemented solution #3 above, probably transitioning to #2 down the line (probably without compression)
String hcc_integration_testing( String inside_args, String job, String build_config )
{
  // Attempt to make unique docker image names for each build, to support concurrent builds
  // Mangle docker org name with upstream build info
  String testing_org_name = 'hcc-test-' + get_upstream_build_project( ).replaceAll('/','-').toLowerCase( ) + '-' + get_upstream_build_num( )
  // String testing_org_name = 'hcc-test-artifacts-download'

  // Tag image name with this build number
  String hcc_test_image_name = "hcc:${env.BUILD_NUMBER}"

  def hip_integration_image = null

  dir( 'integration-testing' )
  {
    deleteDir( )

    // This invokes 'copy artifact plugin' to copy archived files from upstream build
    step([$class: 'CopyArtifact', filter: 'build/**/*.deb, docker/dockerfile-hcc-lc-*',
      fingerprintArtifacts: true, projectName: get_upstream_build_project( ), flatten: true,
      selector: [$class: 'TriggeredBuildSelector', allowUpstreamDependencies: false, fallbackToLastSuccessful: false, upstreamFilterStrategy: 'UseGlobalSetting'],
      target: '.' ])
    // step([$class: 'CopyArtifact', filter: 'build/**/*.deb, docker/dockerfile-hcc-lc-*',
    //   fingerprintArtifacts: true, projectName: 'kknox/hcc/test-artifact-download', flatten: true,
    //   selector: [$class: 'LastCompletedBuildSelector'],
    //   target: '.' ])

    docker.build( "${testing_org_name}/${hcc_test_image_name}", "-f dockerfile-hcc-lc-ubuntu-16.04 ." )
  }

  // Checkout source code, dependencies and version files
  String source_hip_rel = checkout_and_version( job )

  // Conctruct a binary directory path based on build config
  String build_hip_rel = build_directory_rel( build_config );

  // Build hip inside of the build environment
  hip_integration_image = docker_build_image( job, testing_org_name, '', source_hip_rel, "${testing_org_name}/${hcc_test_image_name}" )

  docker_build_inside_image( hip_integration_image, inside_args, job, '', build_config, source_hip_rel, build_hip_rel )

  docker_clean_images( testing_org_name, '*' )
}

////////////////////////////////////////////////////////////////////////
// -- MAIN
// Following this line is the start of MAIN of this Jenkinsfile
String build_config = 'Release'
String job_name = env.JOB_NAME.toLowerCase( )

// Integration testing is a special path which implies testing of an upsteam build of hcc,
// but does not need testing across older builds of hcc or cuda.  This is more of a compiler
// hcc unit test
//  params.hcc_integration_test is set in HCC build
if( params.hcc_integration_test )
{
  println "HCC integration testing"

  node('docker && rocm')
  {
    hcc_integration_testing( '--device=/dev/kfd', 'hcc-ctu', build_config )
  }

  return
}

// The following launches 3 builds in parallel: hcc-ctu, hcc-1.6 and cuda
parallel hcc_ctu:
{
  node('docker && rocm')
  {
    String hcc_ver = 'hcc-ctu'
    String from_image = 'compute-artifactory:5001/radeonopencompute/hcc/clang_tot_upgrade/hcc-lc-ubuntu-16.04:latest'
    String inside_args = '--device=/dev/kfd'

    // Checkout source code, dependencies and version files
    String source_hip_rel = checkout_and_version( hcc_ver )

    // Create/reuse a docker image that represents the hip build environment
    def hip_build_image = docker_build_image( hcc_ver, 'hip', ' --pull', source_hip_rel, from_image )

    // Print system information for the log
    hip_build_image.inside( inside_args )
    {
      sh  """#!/usr/bin/env bash
          set -x
          /opt/rocm/bin/rocm_agent_enumerator -t ALL
          /opt/rocm/bin/hcc --version
        """
    }

    // Conctruct a binary directory path based on build config
    String build_hip_rel = build_directory_rel( build_config );

    // Build hip inside of the build environment
    docker_build_inside_image( hip_build_image, inside_args, hcc_ver, '', build_config, source_hip_rel, build_hip_rel )

    // After a successful build, upload a docker image of the results
    String hip_image_name = docker_upload_artifactory( hcc_ver, job_name, from_image, source_hip_rel, build_hip_rel )

    if( params.push_image_to_docker_hub )
    {
      docker_upload_dockerhub( job_name, hip_image_name, 'rocm' )
      docker_clean_images( 'rocm', hip_image_name )
    }
    docker_clean_images( job_name, hip_image_name )
  }
},
hcc_1_6:
{
  node('docker && rocm')
  {
    String hcc_ver = 'hcc-1.6'
    String from_image = 'compute-artifactory:5001/radeonopencompute/hcc/roc-1.6.x/hcc-lc-ubuntu-16.04:latest'
    String inside_args = '--device=/dev/kfd'

    // Checkout source code, dependencies and version files
    String source_hip_rel = checkout_and_version( hcc_ver )

    // Create/reuse a docker image that represents the hip build environment
    def hip_build_image = docker_build_image( hcc_ver, 'hip', ' --pull', source_hip_rel, from_image )

    // Print system information for the log
    hip_build_image.inside( inside_args )
    {
      sh  """#!/usr/bin/env bash
          set -x
          /opt/rocm/bin/rocm_agent_enumerator -t ALL
          /opt/rocm/bin/hcc --version
        """
    }

    // Conctruct a binary directory path based on build config
    String build_hip_rel = build_directory_rel( build_config );

    // Build hip inside of the build environment
    docker_build_inside_image( hip_build_image, inside_args, hcc_ver, '', build_config, source_hip_rel, build_hip_rel )

    // Not pushing hip-hcc-1.6 builds at this time; saves a minute and nobody needs?
    // String hip_image_name = docker_upload_artifactory( hcc_ver, job_name, from_image, source_hip_rel, build_hip_rel )
    // docker_clean_images( job_name, hip_image_name )
  }
},
nvcc:
{
  node('docker && cuda')
  {
    ////////////////////////////////////////////////////////////////////////
    // Block of string constants customizing behavior for cuda
    String nvcc_ver = 'nvcc-8.0'
    String from_image = 'nvidia/cuda:8.0-devel'

    // This unfortunately hardcodes the driver version nvidia_driver_375.74 in the volume mount.  Research if a way
    // exists to get volume driver to customize the volume names to leave out driver version
    String inside_args = '''--device=/dev/nvidiactl --device=/dev/nvidia0 --device=/dev/nvidia-uvm --device=/dev/nvidia-uvm-tools
        --volume-driver=nvidia-docker --volume=nvidia_driver_375.74:/usr/local/nvidia:ro''';

    // Checkout source code, dependencies and version files
    String source_hip_rel = checkout_and_version( nvcc_ver )

    // We pull public nvidia images
    def hip_build_image = docker_build_image( nvcc_ver, 'hip', ' --pull', source_hip_rel, from_image )

    // Print system information for the log
    hip_build_image.inside( inside_args )
    {
      sh  """#!/usr/bin/env bash
          set -x
          nvidia-smi
          nvcc --version
        """
    }

    // Conctruct a binary directory path based on build config
    String build_hip_rel = build_directory_rel( build_config );

    // Build hip inside of the build environment
    docker_build_inside_image( hip_build_image, inside_args, nvcc_ver, "-DHIP_NVCC_FLAGS=--Wno-deprecated-gpu-targets", build_config, source_hip_rel, build_hip_rel )
  }
}
