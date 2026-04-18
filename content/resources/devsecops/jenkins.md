---
title: "Jenkins"
date: 2026-01-20
---


## 1. Few Questions
### 1.1 Can you integrate the jenkins with the Bitbucket and create the multibranch pipeline ? 
- Integrating your on-presise bitbucket server with Jenkins is great move, allows jenkins to automatically detect new branches, trigger builds when developers push code, and report build status (pass/fail) directly back to Bibucket PR UI.
- Here is Straight forward way to getting Bitbucket and jenkins talking to each other natively.
#### 1.1.1 Generate a token in Bitbucket server
  - Create Bot account with minimum permissions (alteast read, if possible write operation to allow jenkins to put green or red checkmarks next to our commits in Bitbucket)
  - Create the Personal access token with above account, name it something like `jenkins-bot`
  - Copy the token and keep it somewhere safe.
#### 1.1.2 Install required plugins in the Jenkins
    - Install Plugins like `Bitbucket`, `Bitbucket Branch Source Plugin` Using Jenkins plugin manager.
    - Restart the Jenkins
#### 1.1.3 Add Token to Jenkins credentials
    - Go to `manage jenkins` -> `credentials`
    - Click into `System` store, then `Global credentials`, click `add credentials`
    - For the Kind, select `username with password`
      - Username: Bitbucket bot useranme
      - Password; paster the Personal access token you have copied earlier.
      - ID/Description: Name it `bitbucket-server-token`, so its easy to find later.
      - Click `Create`
#### 1.1.4 Configure the Server Connection
  - Lets tell jenkins where your bitbucket server lives
  - Go to `Maanage` -> `System` -> `Bitbucket endpoints` -> `Add` -> `Bitbucket server`
    - Enter name and your server URL (exampel, https://bitbucket.xxx.local)
    - From the dropdown menu, select the credentials you just  created.
    - Click `Test Connection`. If everything is correct, you will see a success message. If it is, great.
  - but from my personal experiance, most of the time, if your on-premise bitbucker server's TLS/SSL configuration has done with self-signed (internal CA) certificate, the Java environment that runs the jenkins say *I dont know who signed this cerificate, so i refuse to talk to this server.* and it will fail with below error during the connection test.
    ```
    PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
    ```
    - Because the jenkins plugin uses java to make API calls to bitbucket, we have to tell jenkins underlying java runtime to trust our self signed certificate. Lets inject this certificate into jenkins java truststore.
    ```bash
    # Download the bitbucket public certficate and save it jenkins server.
    openssl s_client --connect bitbucket.xxx.local:443 < /dev/null 2>/dev/null | openssl x509 -outform PEM > bitbucket-cert.pem

    # Locate the jenkins Java keystore
    # Look at the path to `java` executable in the jenkins process, just like `/usr/lib/jvm/java-17-openjdk-amd64/bin/java`
    # in this case, the java keystore is at `/usr/lib/jvm/java-17-openjdk-amd64/lib/security/cacerts`
    ps -ef | grep jenkins

    # Import the bitbucket certificate to the java keystore using `keytool`
    # Please note that the default password for the java keystore is `changeit`
    # It will prompt you for keystore password, Type: chageit
    sudo keytool -importcert -alias bitbucket-internal-cert -keystore /usr/lib/jvm/java-17-openjdk-amd64/lib/security/cacerts -file bitbucket-cert.pem

    # lets restart jenkins
    sudo systemctl restart jenkins
    ```
  - Fixing the javakeystore, resolves the API communication issues, and jenkins plugin finally see your bitbucket branches.
  - However, when the pipeline actually runs and tries to execute `git clone`, the Git CLI on server might also rejects the certificate because GIT uses the OS-level trust, not Java's anymore.
    - **Quick fix with UI**
    - To fix this, we will force the jenkins to apply equivalnet of `http.sslVerify false` across the board without messing with the command lines on the server.
    - Go to Manage -> System -> Global properties -> Environment variables
    - Add a new variable with name `GIT_SSL_NO_VERIFY` and value `true`, and save it.
    - **Proper Fix with CA Bundle**
    - if its debian based linux
    - `sudo cp bitbucket-cert.pem /usr/local/share/ca-certificates/`
    - `sudo update-ca-certificates`
    - if its redhat based linux
    - `sudo cp bitbucket-cert.pem /etc/pki/ca-trust/source/anchors/`
    - `sudo update-ca-trust extract`
    - yeah, thats it, restart the server with `systemctl restart jenkins`
    - **Permanent fix**
    - `sudo systemctl edit jenkins`
    - Add the following content: `Environment="GIT_SSL_NO_VERIFY=true"` under `[Service]` section
    - `sudo systemctl daemon-reload`
    - `sudo systemctl restart jenkins`
#### 1.1.5 Create Multi-branch Pipeline in Jenkins
  - Go to Jenkins -> New Item -> Multi-branch Pipeline -> Give a name -> Create it.
  - Add bitbucket as the source: 
    - Click add source -> Bitbucket (Assuming you have instlalled required plugins earlier)
    - Now you may see a dropdown for your bitbucket server, select it.
    - Just type in your project key and select your repository from the dropdown.
  - Now, to avoid the infinite loop of branch discovery, we need to configure the scan and git clone strategies.
    - Now look for Behaviours selection, just below repository name, and click `Add` button there.
    - Lets add `Discover branches` trait, and that set to `All branches`,not just pull requests.
    - Select the `Filter by name (with wildcards)` option, and enter `main` or `master` in the include box, and leave the exclude box empty for the time being.
  - click and save the configuration.
#### 1.1.6 Create Jenkinsfile and now push it to the repository
  - Create a `Jenkinsfile` in the root of your repository and push it to `main` branch.
    ```groovy
    pipeline {
        agent any
        stages {
            stage('Run tests') {
                steps {
                    // if you have make target for tests, just run it exactly as you would from your terminal
                    sh 'make run-tests'
                }
            }
        stage('Build Artifact') {
            steps {
                // again, just trigger your build tool
                sh 'make build'
            }
        }
        stage('Publish to artifactory') {
            steps {
                // grabs any jar file from target directory, and save it as an artifact in jenkins
                archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: false

                // lets also push it to artifactory
                withCredentials([usernamePassword(credentialsId: 'artifactory-credentials', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                    sh '''
                    curl -u $ARTIFACTORY_USER:$ARTIFACTORY_PASSWORD -X PUT "http://artifactory.example.com/artifactory/my-repo/my-app.jar" -T target/my-app.jar
                    '''
                }
            }
        }
        }
        post {
            always {
                // clean up any artifacts or temporary files
                cleanWs()
            }
        }
    }
    ```
#### 1.1.7 Trigger the pipeline
  - Go to your multi-branch pipeline and click `Build Now` to trigger the pipeline.






### 1.2 Ok, do you aware how do we configure LDAP authentication in Jenkins? 
- To configure LDAP authentiacation, we need the official, core plugin simply named `LDAP`, please download it from jenkins plugin manager.
- **CRITICAL WARNING:** Keep current jenkins browser open and do this in second tab. if you configure LDAP incorrectly and hit save, you will instantly lock yourself out of jenkins.
- Go to Manage Jenkins --> Security --> Under "Authentication" section, change it from "Jenkins' own user database" to "LDAP".
- Server: Your LDAP server URL. eg: `ldap://ldap.example.com:389` (or `ldaps://ldap.example.com:636` for SSL)
- Configuring search rules:
  - root DN: Base of your directory tree. eg: `dc=example,dc=com`
  - User search base: this is the folder where users live, leave it blank if they are just sitting on root, but usually they are in an organizational unit (OU). eg: `ou=users` ( or `ou=people` )
  - User search filter: Search filter to find users. eg: `uid={0}` (or `cn={0}` if directory uses `cn` for usernames instead of `uid`)
- Setup Manager: Jenkins need account to search the directlory before it can log anyone in.
  - Manager DN: Service account to search the directory. eg: `cn=admin,dc=example,dc=com`
  - Manager Password: Password for the service account. eg: `admin_password`
- Run Test correctly:
  - Username: do not type `cn=admin...`, just type short username,if you want to test the admin, just type `admin` (assuming filter is `cn={0}`).
  - Password: Password for the user you typed above.
  - Hit test again, Because you provided the Manager DN, jenkins will log in securely in background, search for `cn=admin`, find their real DN, and then test their password. 
  - You should see a green sucess with their group membership.


### 1.3 Do you aware how do we clone a multi-branch pipeline job in Jenkins ?
- There were actually few ways to do this, ranging from a hidden UI feature to the exact export/iport trick.
#### 1.3.1 The Hidden Clone Button:
  - Jenkins actually does have a built in clone feature, but they did hid it in most unintuitive place possible. you dont click on your existing job to clone it, instead you have to pretend you are making a new one.
  - Go to Dashboard --> New Item --> Give it a name
  - Ignore all teh project types in middle, instead scroll to the absolute bottom of the page, you will a see text box labeled "Copy from".
  - Start typing the exact name of the existing good multibranch pipeline, it will auto complete.
  - select it and click ok.
  - Jenkins will instantly duplicate the entire configuration of your exising pipeline into new one. All you have to do this, is change the bitbucket repository name, hit save, and done.
#### 1.3.2 The Export/Import Hack (The XML way):
  - Underneath the ugly UI, every single jenkins job is literally a single file called `config.xml`, we can downlaod it, change the repository name, upload it back to create a new job.
  - Go to your working multibranch pipeline job page.
  - Look at the URL in your address bar (eg: https://your-jenkins/job/my-awesome-pipeline/)
  - Simply add `config.xml` to the end of the URL (eg: https://your-jenkins/job/my-awesome-pipeline/config.xml)
  - It will display the raw XML code, save it locally in your computer. lets say `new-awesome-pipeline.xml`
  - Open `new-awesome-pipeline.xml`, change the repository name, and replace it with new one, and save it.
  - Lets use curl to push it back to jenkins as a brand-new job. Once done, it just boom, your new job is ready to go.
  ```bash
  curl -X POST "https://your-jenkins/createItem?name=new-awesome-pipeline" \
    -u "admin:admin_api_token" \
    --data-binary "@new-awesome-pipeline.xml" \
    -H "Content-Type: application/xml"
  ```
#### 1.3.3 The Jenkins Script Console (Groovy way):
  - If you have access to the Jenkins Script Console, you can use a groovy script to clone the job.
  - Go to `Manage Jenkins` --> `Script Console`
  - Run the following script:
  ```groovy
  def sourceJob = jenkins.model.Jenkins.instance.getItem("my-awesome-pipeline")
  def newJob = jenkins.model.Jenkins.instance.copy(sourceJob, "new-awesome-pipeline")
  newJob.save()
  ```
  - This will create a new job with the same configuration as the source job. All you have to do this, is change the bitbucket repository name, hit save, and done.
#### 1.3.4 The Jenkins CLI (Command Line way):
  - If you have access to the Jenkins CLI, you can use it to clone the job.
  - Download the jenkins-cli.jar from your jenkins server.
  - Run the following command:
  ```bash
  java -jar jenkins-cli.jar -s https://your-jenkins/ -auth admin:admin_api_token get-job my-awesome-pipeline > new-awesome-pipeline.xml
  ```
  - Open `new-awesome-pipeline.xml`, change the repository name, and replace it with new one, and save it.
  - Run the following command:
  ```bash
  java -jar jenkins-cli.jar -s https://your-jenkins/ -auth admin:admin_api_token create-job new-awesome-pipeline < new-awesome-pipeline.xml
  ```
  - This will create a new job with the same configuration as the source job. All you have to do this, is change the bitbucket repository name, hit save, and done.
  
#### 1.3.5 The DSL way, using Jenkins Job DSL plugin:
  - If you have the Jenkins Job DSL plugin installed, you can use it to clone the job with a one click.
  - Go to Dashboard --> New Item --> Enter the name of job --> Select Free-Style-Project --> Click on "OK"
  - Scroll down to build steps section, click `add build step` and select `Process Job DSLs`
  - Select the radio button for `Use the provided DSL script`
  - Paste the following DSL script. I only referenced the config.xml file from the source job to write below groovy code.
  ```groovy
  multibranchPipelineJob('new-awesome-pipeline') {
    description('Cloned from my-awesome-pipeline')
    displayName('New Awesome Pipeline')

    // where to find the jenkins file in branch ('factory' XML block)
    factory {
      workflowBranchProjectFactory {
        scriptPath('Jenkinsfile')
      }
    }

    // Dead branch clean up ('orphanedItemStrategy XML Block)
    orphanedItemStrategy {
      discardedItemStrategy {
        daysToKeep(30)
        numToKeep(10)
      }
    }

    // The Bitbucket configuration ('sources' xml block)
    configure { node ->
      // We tell the DSL to navigate to the 'sources/data/jenkins.branch.BranchSource' element
      def branchSource = node / 'sources' / 'data' / 'jenkins.branch.BranchSource'

      // We inject the bitbucket plugin class
      branchSource / 'source'(class: 'com.atlassian.bitbucket.jenkins.internal.scm.BitbucketSCMSource') {
        id('1')
        serverId('c8b2e4f6-7a9d-4c3e-8f2a-1b3c4d5e6f7g')
        credentialsId('bitbucket-credentials')
        projectKey('PROJ')
        repositoryName('new-awesome-repo')

        // We inject the Branch Traits and Filters
        traits {
          'com.atlassian.bitbucket.jenkins.internal.scm.trait.BitbucketSCMSourceDefaultsTrait'()
          'jenkins.scm.impl.trait.WildcardSCMHeadFilterTrait' {
            includes('main', 'develop')
            excludes('feature/*', 'bugfix/*')
          }

          // We inject shallow clone and no tags settings
          'jenkins.plugins.git.traits.CloneOptionTrait' {
            extension(class: 'hudson.plugins.git.extensions.impl.CloneOption') {
              shallow('true')
              noTags('true')
              depth('1')
              honorRefspec('false')
            }
          }
        }
      }

      // Final strategy block requirement
      branchSource / 'strategy'(class: 'jenkins.branch.DefaultBranchPropertyStrategy') {
        properties(class: 'empty-list')
      }
    }
  }
  ```












