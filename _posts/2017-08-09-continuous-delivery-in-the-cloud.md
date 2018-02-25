---
layout: post
title: Continuous delivery in the cloud
tags: [Architecture, Continuous Delivery, DevOps]
gh-repo: ivans-innovation-lab/my-company-monolith
gh-badge: [star, fork, follow]
---

The idea of automated deployment is important. Indeed, if you take automating the deployment process to its logical conclusion, you could push every build that passes the necessary automated tests into production. The practice of automatically deploying every successful build directly into production is generally known as Continuous Deployment.

However, a pure Continuous Deployment approach is not always appropriate for everyone. For example, many users would not appreciate new versions falling into their laps several times a week, and prefer a more predictable \(and transparent\) release cycle. Commercial and marketing considerations might also play a role in when a new release should actually be deployed. The notion of Continuous Delivery is a variation on the idea of Continuous Deployment that takes into account these considerations.

The key pattern introduced in continuous delivery is the _deployment pipeline._

In the deployment pipeline pattern, every change in version control triggers a process \(usually in a [CI](https://continuousdelivery.com/foundations/continuous-integration/) server\) which creates deployable packages and runs automated unit tests and other validations such as static code analysis. This first step is optimized so that it takes only a few minutes to run. If this initial commit stage fails, the problem must be fixed immediately—nobody should check in more work on a broken commit stage. Every passing commit stage triggers the next step in the pipeline, which might consist of a more comprehensive set of automated tests. Versions of the software that pass all the automated tests can then be deployed on demand to further stages such as exploratory testing, performance testing, staging, and production, as shown below.

## Deployment pipeline

**This is a real-life, open-source, java, maven, spring-boot [application](https://github.com/ivans-innovation-lab/my-company-monolith)** with monolithic architectural style. Domain Driven Design is applied through Event Sourcing and CQRS. How Event Sourcing enables deployment flexibility - the monolithic application can be migrated and deployed as a microservices. You can find more information about the structure of application/s [here](http://ivans-innovation-lab.github.io/projects).

The infrastructure and tools needed for the pipeline are:

* [Artifactory](https://www.jfrog.com/artifactory/) as Maven repository on AWS
  * [http://maven.idugalic.pro](http://maven.idugalic.pro)
* [CircleCI](https://circleci.com/) as continuous integration and delivery platform
* [PWS](http://run.pivotal.io/) - an instance of the Cloud Foundry platform-as-a-service operated by Pivotal Software, Inc.


This "Pipeline as Code" is written to a [.circleci/config.yml](https://github.com/ivans-innovation-lab/my-company-monolith/blob/master/.circleci/config.yml) and checked into a project’s source control repository:


```
defaults: &defaults
  working_directory: /home/circleci/my-company-monolith
  docker:
    - image: circleci/openjdk:8-jdk-browsers
    
version: 2
jobs:
  # Build and test with maven
  build:
    <<: *defaults
    steps:

      - checkout
      # Caching is one of the most effective ways to make jobs faster on CircleCI.
      # Downloading from a Remote Repository in Maven is triggered by a project declaring a dependency that is not present in the local repository (or for a SNAPSHOT, when the remote repository contains one that is newer).
      # Do not overwrite your release (not snapshots) artifacts (my-company-blog-domain, my-company-blog-materialized-view, my-company-project-domain, my-company-project-materialized-view) on remote maven repository, othervise the cache will become stale.
      - restore_cache:
          key: my-company-monolith0003-{{ checksum "pom.xml" }}

      - run: 
          name: Build and Install maven artifact
          command:  mvn -s .circleci/maven.settings.xml install -P idugalic-cloud

      - save_cache:
          paths:
            - ~/.m2
          key: my-company-monolith0003-{{ checksum "pom.xml" }}
      
      - run:
          name: Collecting test results
          command: |
            mkdir -p junit/
            find . -type f -regex ".*/target/surefire-reports/.*xml" -exec cp {} junit/ \;
          when: always
          
      - store_test_results:
          path: junit/
          
      - store_artifacts:
          path: junit/
          
      - run:
          name: Collecting artifacts
          command: |
            mkdir -p artifacts/
            cp pom.xml artifacts/
            cp .circleci/maven.settings.xml artifacts/
            find . -type f -regex ".*/target/.*jar" -exec cp {} artifacts/ \;
     
      - store_artifacts:
          path: artifacts/
          
      - persist_to_workspace:
          root: artifacts/
          paths:
            - .
   
  # Deploying build artifact on PWS staging environment for testing.
  staging:
    <<: *defaults
    steps:
      - attach_workspace:
          at: workspace/
      - run:
          name: Install CloudFoundry CLI
          command: |
            curl -v -L -o cf-cli_amd64.deb 'https://cli.run.pivotal.io/stable?release=debian64&source=github'
            sudo dpkg -i cf-cli_amd64.deb
            cf -v
      - deploy:
          name: Deploy to Staging - PWS CLoudFoundry (CF_PASSWORD variable required)
          command: |
            cf api https://api.run.pivotal.io
            cf auth idugalic@gmail.com $CF_PASSWORD
            cf target -o idugalic -s Stage
            cf push stage-my-company-monolith -p workspace/*.jar --no-start
            cf bind-service stage-my-company-monolith mysql
            cf restart stage-my-company-monolith
  
  # Deploying current/old production on PWS staging environemnt for DB schema backward compatibility testing.
  production-on-staging:
    <<: *defaults
    steps:
      - attach_workspace:
          at: workspace/
          
      - run:
          name: Install CloudFoundry CLI
          command: |
            curl -v -L -o cf-cli_amd64.deb 'https://cli.run.pivotal.io/stable?release=debian64&source=github'
            sudo dpkg -i cf-cli_amd64.deb
            cf -v
      
      - run: 
          name: Install AWS CLI
          command: sudo apt-get update && sudo apt-get install -y awscli

      - run:
          name: Download latest production artifact from AWS S3 (AWS Permissions on CircleCI required)
          command: aws s3 sync s3://my-company-production . --region eu-central-1
          
      - deploy:
          name: Deploy latest production application to Staging - PWS CLoudFoundry (CF_PASSWORD variable required)
          command: |
            if [ -e "./my-company-monolith-1.0.0-SNAPSHOT.jar" ]; then
              cf api https://api.run.pivotal.io
              cf auth idugalic@gmail.com $CF_PASSWORD
              cf target -o idugalic -s Stage
              cf push stage-my-company-monolith-prod -p workspace/*.jar --no-start
              cf bind-service stage-my-company-monolith-prod mysql
              cf restart stage-my-company-monolith-prod 
            else 
              echo "Artifact does not exist, deploying current build"
              cf api https://api.run.pivotal.io
              cf auth idugalic@gmail.com $CF_PASSWORD
              cf target -o idugalic -s Stage
              cf push stage-my-company-monolith-prod -p workspace/*.jar --no-start
              cf bind-service stage-my-company-monolith-prod mysql
              cf restart stage-my-company-monolith-prod
            fi 

  
  # A very simple e2e test on PWS staging environemnt
  staging-e2e:
    <<: *defaults
    steps:
      - run: 
          name: End to end test on Staging
          command: |
            curl -I "https://stage-my-company-monolith.cfapps.io/health"
            curl -I "https://stage-my-company-monolith.cfapps.io/api/blogposts"
            curl -I "https://stage-my-company-monolith.cfapps.io/api/projects"
      
  # Testing if the latest DB schema is compatible with latest production application (DB schema backward compatibility testing).
  production-on-staging-e2e:
    <<: *defaults
    steps: 
      - run: 
          name: End to end test on Staging of current production
          command: |
            curl -I "https://stage-my-company-monolith-prod.cfapps.io/health"
            curl -I "https://stage-my-company-monolith-prod.cfapps.io/api/blogposts"
            curl -I "https://stage-my-company-monolith-prod.cfapps.io/api/projects"
              
  # Deploying build artifact on PWS production environment with Blue-Green deployment strategy and the rollback option.
  # Build artifact is uploaded to AWS s3 as the latest production artifact in this stage. It is used within 'production-on-staging' and 'production-on-staging-e2e' jobs to test DB schema backward compatibility
  production:
    <<: *defaults
    steps:
      - attach_workspace:
          at: workspace/
          
      - run:
          name: Install CloudFoundry CLI
          command: |
            curl -v -L -o cf-cli_amd64.deb 'https://cli.run.pivotal.io/stable?release=debian64&source=github'
            sudo dpkg -i cf-cli_amd64.deb
            cf -v
            
      - run: 
          name: Install AWS CLI
          command: sudo apt-get update && sudo apt-get install -y awscli
          
      - deploy:
          name: Deploy to Production - PWS CLoudFoundry (CF_PASSWORD variable required)
          command: |
            cf api https://api.run.pivotal.io
            cf auth idugalic@gmail.com $CF_PASSWORD
            cf target -o idugalic -s Prod
            
            cf push prod-my-company-monolith-B -p workspace/*.jar --no-start
            cf bind-service prod-my-company-monolith-B mysql
            cf restart prod-my-company-monolith-B
            # Map Original Route to Green
            cf map-route prod-my-company-monolith-B cfapps.io -n prod-my-company-monolith
            # Unmap temp Route to Green
            cf unmap-route prod-my-company-monolith-B cfapps.io -n prod-my-company-monolith-B
            # Unmap Route to Blue (only if Blue exist)
            cf app prod-my-company-monolith && cf unmap-route prod-my-company-monolith cfapps.io -n prod-my-company-monolith
            # Stop and rename current Blue in case you need to roll back your changes (only if Blue exist)
            cf app prod-my-company-monolith && cf stop prod-my-company-monolith
            cf app prod-my-company-monolith-old && cf delete prod-my-company-monolith-old -f
            cf app prod-my-company-monolith && cf rename prod-my-company-monolith prod-my-company-monolith-old
            # Rename Green to Blue
            cf rename prod-my-company-monolith-B prod-my-company-monolith
                           
      - deploy:
          name: Upload latest production artifact to AWS S3 (AWS Permissions on CircleCI required)
          command: aws s3 sync workspace/ s3://my-company-production --delete --region eu-central-1
          
notify:
  webhooks:
    - url: https://webhook.atomist.com/atomist/circle
  
workflows:
  version: 2
  my-company-monolith-workflow:
    jobs:
      - build
      - staging:
          requires:
            - build
          filters:
            branches:
              only: master
      - staging-e2e:
          requires:
            - staging
          filters:
            branches:
              only: master
      - production-on-staging:
          requires:
            - staging-e2e
          filters:
            branches:
              only: master
      - production-on-staging-e2e:
          requires:
            - production-on-staging
          filters:
            branches:
              only: master
      - approve-production:
          type: approval
          requires:
            - production-on-staging-e2e
          filters:
            branches:
              only: master
      - production:
          requires:
            - approve-production
          filters:
            branches:
              only: master
```

The following example shows a [pipeline](https://circleci.com/gh/ivans-innovation-lab/workflows/my-company-monolith) with seven jobs (steps). The jobs run according to configured requirements, each job waiting to start until the required job finishes successfully. This pipeline is configured to wait for manual approval of a job 'approve-production' before continuing by using the `type: approval` key. The `type: approval` key is a special job that is only added under your `workflow` key

![](https://github.com/idugalic/idugalic.github.io/raw/master/img/Screen%20Shot%202018-02-25%20at%205.41.17%20PM.png)

![](https://github.com/idugalic/idugalic.github.io/raw/master/img/CIrcleCI-Monolith.png)

### Build 

Application is build on every push to git repository. Build can be triggered from any branch or pull request. 'Build' job is using maven to build and test the application. Successful build will create an artifact (jar file) that is shared with all the jobs in the pipeline.

### Staging

Every merge or push to **master** branch will trigger the pipeline and the artifact will be deployed to PWS on '**Stage**' space:![](https://docs.lab.idugalic.pro/assets/Screen%20Shot%202017-06-21%20at%201.28.42%20PM.png) Additionally, a current production artifact will be deployed on Stage by 'production-on-staging' job for DB schema backward compatibility testing \(we will test old application against new DB schema\). This will enable Blue-Green deployment with roll-back option, as shown below under [Blue-Green Deployment section](#blue-green-deployment).

### Production

Once you are ready to deploy to production you should **manually approve deployment to production** in you CircleCI workflow/pipeline. This will trigger then next job \('production'\), and the application will be deployed (with zero-downtime) to PWS on '**Prod**' space:![](https://docs.lab.idugalic.pro/assets/Screen%20Shot%202017-06-21%20at%201.28.58%20PM.png) You can consider:

 - removing manual step ('approve') and practice Continuous Deployment instead of Continuous Delivery.
 - introduce new branch 'production', and filter 'production' job based on it. This way you can deploy to production by merging 'master' to 'production'.
 - [filter 'production' job by git tags](https://circleci.com/docs/2.0/workflows/#git-tag-job-execution). This way you can deploy to production by creating a new tag (and release) on your git repository. CircleCI treats tag and branch filters differently when deciding whether a job should run: for a branch push unaffected by any filters - CircleCI runs the job, for a tag push unaffected by any filters - CircleCI skips the job.

### Requirements

For the pipeline to work you have to create two spaces \(environments\) on PWS:

* Stage
* Prod

On each space you have to create instance of ClearDB MySQL service \(database\):

```
cf api https://api.run.pivotal.io
cf auth EMAIL PASSWORD
cf target -o idugalic -s Stage
cf create-service cleardb spark mysql
cf t -s Prod
cf create-service cleardb spark mysql
```

NOTE: Instructions to install CloudFoundry CLI \(cf\): [https://docs.cloudfoundry.org/cf-cli/install-go-cli.html](https://docs.cloudfoundry.org/cf-cli/install-go-cli.html)

### Metrics

PCF Metrics helps you understand and troubleshoot the health and performance of your apps by displaying the following:

* [Container Metrics](http://docs.run.pivotal.io/metrics/using.html#container)
   A graph of CPU, memory, and disk usage percentages
* [Network Metrics](http://docs.run.pivotal.io/metrics/using.html#network)
   A graph of requests, HTTP errors, and response times
* [App Events](http://docs.run.pivotal.io/metrics/using.html#events)
   A graph of update, start, stop, crash, SSH, and staging failure events
* [Logs](http://docs.run.pivotal.io/metrics/using.html#logs)
   A list of app logs that you can search, filter, and download
* [Trace Explorer](http://docs.run.pivotal.io/metrics/using.html#trace)
   A graph that traces a request as it flows through your apps and their endpoints, along with the corresponding logs

![](https://docs.lab.idugalic.pro/assets/Screen%20Shot%202017-06-16%20at%2011.45.31%20AM.png)

### Autoscaler

[App Autoscaler](https://docs.run.pivotal.io/appsman-services/autoscaler/using-autoscaler.html) is a marketplace service that ensures app performance and helps control the cost of running apps.

To balance app performance and cost, Space Developers and Space Managers can use App Autoscaler to do the following:

* Configure rules that adjust instance counts based on metrics thresholds such as CPU Usage
* Modify the maximum and minimum number of instances for an app, either manually or following a schedule

### Spring Boot Actuator

Adding Actuator to your Spring Boot application deployed on Pivotal Cloud Foundry gets you the following production-ready features:

* Health Check column & expanded information in Instances section
* git commit id indicator, navigable to your git repo
* Summary git info under Settings tab \(also navigable to repo\)
* Runtime adjustment of logging levels, exposed via Actuator endpoints
* Heap Dump\*
* View Trace\*
* View Threads, dump/download for further analysis\*

### Blue-Green Deployment

[Blue-green deployment](https://docs.run.pivotal.io/devguide/deploy-apps/blue-green.html) is a release technique that reduces downtime and risk by running two identical production environments called Blue and Green.

At any time, only one of the environments is live, with the live environment serving all production traffic. For this example, Blue is currently live and Green is idle.

As you prepare a new release of your software, deployment and the final stage of testing takes place in the environment that is not live: in this example, Green. Once you have deployed and fully tested the software in Green, you switch the router so all incoming requests now go to Green instead of Blue. Green is now live, and Blue is idle.

This technique can eliminate downtime due to application deployment. In addition, blue-green deployment reduces risk: if something unexpected happens with your new release on Green, you can immediately roll back to the last version by switching back to Blue.

Blue-green deployment is implemented by 'production' job in the [workflow](https://github.com/ivans-innovation-lab/my-company-monolith/blob/master/.circleci/config.yml).

Doing Blue-green deployment with database schema changing is hard. We have to [change the schema](https://martinfowler.com/books/refactoringDatabases.html) in such a way that Blue-green deployment and roll-back to the previous version are possible, usually by making DB changes backward compatible (this makes DB schema backward compatibility testing an important step). For this we need schema versioning first \([Flyway](http://flywaydb.org/)\). I was inspired with this [blog post](https://spring.io/blog/2016/05/31/zero-downtime-deployment-with-a-database).

You can design your database in the 6th normal form an make you scheme more adaptable and your process more agile. I was inspired with this [blog post](https://blog.codecentric.de/en/2017/07/agile-database-design-using-anchor-modeling/ ).

