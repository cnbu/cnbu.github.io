```yaml
pipeline {
  options {
    timeout(time:30, unit: 'MINUTES')
    buildDiscarder logRotator(numToKeepStr: '1')
  }
  parameters {
    string(name: 'app_git_url', defaultValue: 'http://techdoc.oa.com/devops/hcboard.git', description: '' )
    string(name: 'app_git_branch', defaultValue: 'master', description: '' )
    string(name: 'app_dir', defaultValue: '/', description: '' )
    string(name: 'app_name', defaultValue: 'hcboard', description: '' )
    string(name: 'target_dir', defaultValue: '/target', description: '' )
    string(name: 'image_name', defaultValue: 'harbor-test.hzins.com/devops/hcboard:latest', description: '' )
    string(name: 'limit_cpu', defaultValue: '2000m', description: '' )
    string(name: 'limit_memory', defaultValue: '2048Mi', description: '' )
    string(name: 'app_replicas', defaultValue: '1', description: '' )
    string(name: 'jvm_args', defaultValue: '-server -Xmx2g -Xms2g -Xmn256m -XX:PermSize=128m  -XX:+DisableExplicitGC -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:+UseCMSCompactAtFullCollection -XX:LargePageSizeInBytes=128m -XX:+UseFastAccessorMethods -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=70  -Dfile.encoding=utf-8  -Duser.timezone=Asia/Shanghai -Djava.security.egd=file:/dev/./urandom', description: '' )
    string(name: 'env_vars', defaultValue: 'key1=value1,key2=value2,key3=value3', description: '' )
    string(name: 'gitlab_key', defaultValue: 'gitlab-key', description: '' )
    string(name: 'apollo_env', defaultValue: 'DEV', description: '' )
    string(name: 'environment', defaultValue: 'DEV', description: '' )
    string(name: 'namespace', defaultValue: 'ops-dev', description: '' )
  }
  agent {
    kubernetes {
      label "jnlp-slave_${JOB_NAME}_${BUILD_ID}"
      defaultContainer 'jnlp'
      yaml """
        apiVersion: v1
        kind: Pod
        metadata:
          labels:
            some-label: some-label-value
        spec:
          nodeSelector:
            node: public
          containers:
          - name: jnlp
            image: harbor-test.hzins.com/public/jnlp-slave:alpine
            args: ['\$(JENKINS_SECRET)', '\$(JENKINS_NAME)']
          - name: maven
            image: harbor-test.hzins.com/public/base-centos7-jdk1.8.0_201-nodejs-mvn
            imagePullPolicy: IfNotPresent
            command:
            - cat
            tty: true
            volumeMounts:
              - name: mvn
                mountPath: /root/.m2/repository
          - name: docker
            image: harbor-test.hzins.com/public/docker:18.09-dind
            imagePullPolicy: IfNotPresent
            command:
            - cat
            tty: true
            volumeMounts:
              - name: mvn
                mountPath: /root/.m2/repository
              - name: docker-sock
                mountPath: /var/run/docker.sock
              - name: host-time
                mountPath: /etc/localtime
          - name: helm-kubectl
            image: harbor-test.hzins.com/public/alpine-helm-kubectl
            imagePullPolicy: IfNotPresent
            command:
            - cat
            tty: true
            volumeMounts:
              - name: mvn
                mountPath: /root/.m2/repository
          volumes:
          - name: mvn
            nfs:
              path: /mvn-repo
              server: 172.20.0.110
          - name: docker-sock
            hostPath:
              path: /var/run/docker.sock
          - name: host-time
            hostPath:
              path: /etc/localtime
        """
    }
  }
  environment {
    git_ver = "${app_git_branch}"
  }
  stages {
    stage('git scm pull') {
      steps {
        git branch: "${app_git_branch}", credentialsId: "${gitlab_key}", url: "${app_git_url}"
      }
    }
    stage('Build project') {
      steps {
        container('maven') {
          sh '''
            cd .${app_dir}
            mvn clean install -Dmaven.test.skip=true;
            cd -
            rm -f ./${target_dir}/${app_name}*sources.jar
          '''
        }
      }
    }
    stage('Docker-build') {
      steps {
        container('docker') {
          withDockerRegistry(credentialsId: 'registry', url: 'http://harbor-test.hzins.com') {
            // 生成Dockerfile
            sh '''
              echo 'FROM harbor-test.hzins.com/public/base-centos7-jdk1.8.0_201'                                            > .${app_dir}/Dockerfile
              echo 'MAINTAINER admin'                                                                                       >> .${app_dir}/Dockerfile
              echo "WORKDIR /home/deploy/"                                                                                  >> .${app_dir}/Dockerfile
              echo "ADD .${target_dir}/${app_name}*.jar ./app.jar"                                                          >> .${app_dir}/Dockerfile
              echo 'CMD java $JAVA_MEM_OPTS -Dserver.port=9090 -Denv=$APOLLO_ENV -Dapollo.meta=$APOLLO_META -jar app.jar'   >> .${app_dir}/Dockerfile
            '''
            
            // 构建并推送镜像
            sh '''
              docker build --no-cache  -t ${image_name} -f .${app_dir}/Dockerfile .
              docker push  ${image_name}
            '''
          }
        }
      }
    }
    stage('Deploy-app') {
      steps {
        container('helm-kubectl') {
          // 准备k8s认证信息
          sh '''
            mkdir -p /root/.kube/;  cp -rf /root/.m2/repository/kubeconfig/ci-config /root/.kube/config
            kubectl config use-context kubernetes-dev
          '''

          // 生成yaml文件
          sh '''
            #动态生成helm配置和模板文件
            mkdir -p ${namespace}/templates
            echo "name: ${namespace}" >${namespace}/Chart.yaml
            echo "version: 1.0.0" >>${namespace}/Chart.yaml

            cp /root/.m2/repository/kubeconfig/ci-apollo_env_template.yaml ${namespace}/templates
            cd  ${namespace}
            mkdir -p ${app_name}

            #初始化yaml
            helm install . --dry-run --debug \
            --set appname=${app_name} \
            --set replicas=${app_replicas} \
            --set image.repository=${image_name} \
            --set resources.requests.cpu="${limit_cpu}" \
            --set resources.requests.memory="${limit_memory}" \
            --set resources.limits.cpu="${limit_cpu}" \
            --set resources.limits.memory="${limit_memory}" \
            --set APOLLO.ENV="${apollo_env}" \
            --set APOLLO.META='http://apollo-meta.ha.com' \
            --set JAVA_MEM_OPTS="${jvm_args}" \
            --set envs.key1='value1' \
            --set envs.key2='value2' \
            --set envs.key3='value3' \
            | grep -A 1000 'apiVersion' > ${app_name}/${app_name}.yaml
            cat ${app_name}/${app_name}.yaml

            #复制生成后的yaml至nfs主机目录保存，供后绪手动操作
            cp -rf ../${namespace} /root/.m2/repository/k8s-yaml-config/
          '''

          // 部署到K8S
          sh '''
            set +e
            kubectl -n ${namespace} apply -f ${namespace}/${app_name}/
            kubectl get pod,svc -n ${namespace} -o wide
          '''
        }
      }
    }
  }
}
```
