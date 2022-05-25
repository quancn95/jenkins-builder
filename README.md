# Setup environment
====================


**## Setup Jenkin Controller & Agents**
-------------------

* For lab intent, I will use Virtualbox machines and Vagrant to bring them up (recommend use Virtualbox version 6.1.30 - I used Windows 10 as the host and Virtualbox VMs with Ubuntu as guests). 

* In Vagrantfile, I define 3 virtual machines with Ubuntu 18.04 base. One is Controller (running Jenkins controller and Jfrog Artifactory) and the others are for Agents (running Jenkins's tasks).

    * All machines need to install Docker and Java (should be java 8 or 11) inside.

    * On Controller, Jenkins application will run on port 8080 and 50000. With Jfrog will be exposed on port 8081 and 8082. Thus, we need forward these port (8080/50000 & 8081/8082) from Virtualbox for interactive from localhost.

    * To bring the virtual machines up, open terminal on directive contain Vagrantfile and then run command:

      ```
      # Start all VM
      vagrant up
      # After all machines up, you can check vm info by:
      vagrant ssh-config
      ```

**Controller setup**

* First, ssh to the Controller that named `controller-ubuntu-1` and start Jenkins and Jfrog application 
      
        `vagrant ssh controller-ubuntu-1`
            
* Go to the home directory, then start docker containers for Jenkins and Jfrog applications.
    
  Jenkins docker container will act like Jenkins Dashboard and Controller.

  Jfrog Artifactory container is binary repository. We will push artifacts of Jenkins pipeline in.
        
  Note: use this content of [docker-compose.yml](docker-compose.yml) for step 3 below.
  
      ```
      # 1. Move to home directory of ssh user
      cd /home/vagrant
      
      # 2. Create a `docker-compose.yml` file.
      touch docker-compose.yml
      
      # 3. Copy content of "docker-compose.yml" in this repository and paste to docker-compose.yml just created then save.
      vi docker-compose.yml
      
      # 4. Start docker containers of Jenkins and Jfrog
      docker-compose up -d
      ```
      
**Jenkins configure**
   
  * After start Jenkins on previous step, you will be able to access to Jenkins dashboard on your host machine

  * Open browser and go to `http://localhost:8080`. Wait Jenkins boot up and then setup as default with all suggest plugin. Plus:
    
    ```
            + Unity3d plugin
            + Blue Ocean
            + Docker
            + Docker Pipeline
            + Multibranch Scan Webhook Trigger
            + Workspace Cleanup Plugin
    ```
  
  * Next, configure Unity3d tool

    Go to `Dashboard >> Manage Jenkins >> Global Tool Configuration >> Unity3d >> Unity3d installations >> Add Unity3d`
            
    Example add Unity for Ubuntu base then Click **Save**
     - `Name`: *Unnity3d Ubuntu*
     - `Installation directory`: */opt/unity/Editor*

        
  * Configure Credentials
    
    Go to `Dashboard >> Manage Jenkins >> Manage Credentials`.
    
    On `Stores scoped to Jenkins`, chose `Jenkins >> Global credentials`. Then Add credentials for Jenkins pipeline with:
          
     - `unity-email`: *\<Email login to Unity\>* (Kind: _Secret text_)
          
     - `unity-password`: *\<Password login to Unity\>* (Kind: _Secret text_)
          
     - `unity-license-docker-linux`: *\<License file of Unity\>* (Kind: _Secret file_) 
            
       **_Note_**: I will use Docker container base on Linux architecture to run Unity, so I need to activate license manually on Linux machine.
              If you activate on a Windows machine, it may not work correctly with Docker container base Linux.
              ([reference](https://docs.unity3d.com/Manual/ManualActivationGuide.html))
              
              1. On a Linux machine had installed Unity3d, open terminal and run:
              ```
              export UNITY_USERNAME="your username login to Unity"   
              export UNITY_PASSWORD="your password login to Unity"   
              /opt/unity/Editor/Unity -batchmode -nographics -username "$UNITY_USERNAME" -password "$UNITY_PASSWORD" -logFile
              ```
              Then you will receive a file ***.alf**
              
              2. Store the **.alf** for next steps.
              3. Visit [https://license.unity3d.com/manual](https://license.unity3d.com/manual).
              4. Upload **.alf** in the form
              5. Answer questions (unity pro vs personal edition, both will work, just pick the one you use)
              6. Download ***.x.ulf** file (year should match your unity version here, 'Unity_v2018.x.ulf' for 2018, etc.)"
              7. Copy the content of ***.x.ulf** license file to your Jenkins credential variable **unity-license-docker-linux**.
          
     - `jfrog-url`: *\<Ip/Domain of Jfrog Artifactory\>* (Kind: _Secret text_) 
         
       **_Note_**: Because I run Jfrog as docker container and exposed port 8081 & 8082 on Controller VM, so we can use url: `http://127.0.0.1:8081`       
          
     - `jfrog-access-token`: *\<Access token get from Jfrog Artifactory\>* (Kind: _Secret text_)

       **_Note_**: Go to `http://127.0.0.1:8082`, then login to Jfrog with default user/pass is: `admin/password`.
                   On *Administration* tab, chose User Management >> Access Tokens.
                   Click *Generate Token* and then store the token for `jfrog-access-token` on Jenkins
          
     - `github-personal-access-token`: *\<Personal Access Token of Github\>* (Kind: _Secret text_)
              
     - `github-authentication`: *\<Username and Password login to Github\>* (Kind: _Username with password_)
                
       **_Note_**: Github not allow to use **password** get access to Github repository anymore. So please replace the **password** by **Personal Access Token**.
          
     - `agent1-ssh-private-key`: *\<Private key can ssh to the Agent machine\>* (Kind: _SSH Username with private key_)
          
  * Next config Jenkins nodes. Go to `Dashboard >> Manage Jenkins >> Manage nodes and clouds`. We will add new node as an agent.
           
    Example:
        
    * Chose *New Node*:
      - `Node name`: _ubuntu-docker-agent1_
      - `Type`: _Permanent Agent_
      - Click on ***Create***
    
    * Next configure for _ubuntu-docker-agent1_:
      - `Name`: _ubuntu-docker-agent1_
      - `Description`: _Agent ubuntu-docker base_
      - `Number of executors`: _2_
      - `Remote root directory`: _/home/vagrant/jenkins_agent/_
      - `Labels`: _ubuntu docker agent1_
      - `Usage`: _Use this node as much as possible_
      - `Launch method`: _Launch agents via SSH_ (If your agent is a Windows base and it do not have ssh connection. So use can chose _Launch agent by connecting it to the controller_)
              
         - `Host`: _\<The ip of the Agent vm\>_
         - `Credentials`: _\<Chose username & password of **agent1-ssh-private-key**>_
         - `Host Key Verification Strategy`: _Known hosts file Verification Strategy_
            
      - `Node Properties >> Tool Locations`
              
         List of tool locations: 
         - `Name`: _Chose (Unity3d)..._
         - `Home`: _/opt/unity/Editor_
         - Click on **_Save_**  
            
**Setup known host for ssh connection from Jenkins controller to agent**
  
* Back to Controller terminal, move to *jenkins_home* directory. This is volume location is mounted  to jenkins_home inside Docker container(jenkins-controller).
  
  We will add known hosts for Jenkins controller.     
            ```
            # Move to jenkins_home directory
            cd /home/vagrant/jenkins/jenkins_home
            # Create a directory name .ssh
            sudo mkdir .ssh
            # Add known_hosts. Replace you agent's IP
            sudo ssh-keyscan -H <IP of Agent vm> >> .ssh/known_hosts
            ```  


* Relaunch agent       

  On the browser >> `Manage nodes and clouds`, now you chose your node that is created in previous and **_Launch agent_** if you need.  



**## Set up Pipeline**
-------------------

* Back to Jenkins Dashboard on browser, now you are able to create a Pipeline
          
* Chose `New Item >> Enter name of project (ex: 2D-game-platform) >> Multibranch Pipeline >> OK`
          
  Inform then click on **Save**. Example:
  - `Display Name`: _2D-game-platform_
  - `Description`: _CI pipeline for build 2D-game-platform_
  - `Branch Sources >> Git`:

     - `Project Repository`: _\<Enter the url of your Github repository\>_
              
       `Credentials`: _\<chose the `github-authentication` credential(added in step before with username/personal-access-token)>_
  
  - `Scan Multibranch Pipeline Triggers`: _Tick on Scan by webhook_
             
     `Trigger token`: _\<Enter a token like **2D-game-platform-token**>_

     **_Note_**: After that, you will receive the link like so: `JENKINS_URL/multibranch-webhook-trigger/invoke?token=[Trigger token]`.
                      Use that link(replace Jenkins url and Trigger token to the link format) and add it to Github Webhook.

* Add Jenkinsfile for your project.

  - Copy the [Jenkinsfile](Jenkinsfile) to root your project.

  - Copy the [BuildScript.cs](BuildScript.cs) to folder `Assets/Editor/` on your project.

* Save the Multibranch Pipeline and you will be able to run pipeline for each branch of Git repository. 
    Example, go to `2D-game-platform` and let try to launch build on branch `main` by click *Build Now*.

    * Currently, the Pipeline will run for each Windows, MacOS and Android targets. And each of target will include jobs: Activate license, Run Test, Build, Publish artifacts from Build to Jfrog artifactory.
  
* After pipeline finish and pass in all stages, You can see some artifacts are pushed to Jfrog artifactory under `Repository Path: unity-internal/`.

    Artifact will be zipped as `*.tar.gz` file. It contains the Build result of Unity3d.



