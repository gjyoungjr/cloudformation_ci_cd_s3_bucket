In this article, I will give a step-by step guide on how to create a CI/CD Pipeline for a React Web App hosted as a static website in a S3 Bucket. 

Below is an architecture diagram illustrating the AWS services that were used to build this solution.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ystv87ngk5i8dce2m49r.png)

Now that you have reviewed the architecture, let's get straight into it. 

## **<u>Create React App</u>**

If you want to follow along with your own react application please feel free to do so. If not, head over to github and clone the [repo] (https://github.com/dirtbag-ctrl/cloudformation_ci_cd_s3_bucket). 

## **<u>Creating buildspec file </u>**

Once you have the repo cloned, take a minute to review the buildspec.yml file, this file is used to provide AWS CodeBuild instructions on how to build our application. It has 4 stages, install, pre_build, build and post_build and lastly it output our artifacts files.

**<u>Note</u>**: The buildspec file is created at the root level inside your directory.

 

```

version: 0.2

phases:

  install:

    runtime-versions:

      nodejs: 16

  pre_build:

    commands:

      # install dependencies

      - echo installing dependencies...

      - yarn install

  build:

    commands:

      # run build script

      - echo Build started on `date`

      - echo Building React Application...

      - yarn build

  

  post_build:

    commands:

      - echo Build completed on `date`

artifacts:

  # include all files required to run application

  # we include only the static build files

  files:

    - "**/*"

   # reference directory where build file is located

  base-directory: "build"

```

## **<u>Creating CloudFormation Template</u>**

Next, we'll look at our CloudFormation template that provision our resources used to create the pipeline. 

The first thing we do is defined our params that will be used through out our template.

```

AWSTemplateFormatVersion: 2010-09-09

Description: This template is used to create a CI/CD Pipeline that deploys a React Web app to S3 for static website hosting.

# Parameters to be used through out the template

Parameters:

  Stage:

    Type: String

    Default: dev

  AppName: 

    Type: String

    Default: <APP NAME>

  GithubUserName:

    Type: String

    Default: <GITHUB USERNAME>

  GithubRepo:

    Type: String

    Default: <GITHUB REPO>

  GithubBranch:

    Type: String

    Default: <GITHUB BRANCH>

  GithubOAuthToken:

    Type: String

    Default: <GITHUB ACCESS TOKEN>

```

Next we create our policies to give our resources proper permissions.

```

# Create role for CodeBuild

  CodeBuildRole:

    Type: AWS::IAM::Role

    Properties:

      AssumeRolePolicyDocument:

        Version: "2012-10-17"

        Statement:

          - 

            Effect: Allow

            Principal:

              Service:

                - "codebuild.amazonaws.com"

            Action:

              - "sts:AssumeRole"

      Path: /service-role/

      Policies:

        - PolicyName: root

          PolicyDocument:

            Version: "2012-10-17"

            Statement: 

              - 

                Effect: Allow

                Action:

                  - "s3:GetObject"

                  - "s3:GetObjectVersion"

                  - "s3:GetBucketVersioning"

                  - "s3:PutObject"

                  - "s3:PutObjectAcl"

                  - "s3:PutObjectVersionAcl"

                Resource: 

                  - !GetAtt PipelineBucket.Arn

                  - !Join ['', [!GetAtt PipelineBucket.Arn, "/*"]]

              - 

                Effect: Allow

                Action:

                  - "s3:GetObject"

                  - "s3:GetObjectVersion"

                  - "s3:GetBucketVersioning"

                  - "s3:PutObject"

                  - "s3:PutObjectAcl"

                  - "s3:PutObjectVersionAcl"

                Resource: 

                  - !GetAtt DeployBucket.Arn

                  - !Join ['', [!GetAtt DeployBucket.Arn, "/*"]]

              -

                Effect: Allow

                Action:

                  - "logs:CreateLogGroup"

                  - "logs:CreateLogStream"

                  - "logs:PutLogEvents"

                  - "cloudfront:CreateInvalidation"

                Resource:

                  - "*"

      Tags:

        - Key: Name

          Value: !Join ['-', [!Ref AppName, !Ref 'AWS::AccountId', 'BuildRole', !Ref Stage]]

  # Create role for CodePipeline

  CodePipeLineRole:

    Type: AWS::IAM::Role

    Properties:

      AssumeRolePolicyDocument:

        Version: "2012-10-17"

        Statement:

          - 

            Effect: Allow

            Principal:

              Service:

                - "codepipeline.amazonaws.com"

            Action:

              - "sts:AssumeRole"

      Policies:

        - PolicyName: root

          PolicyDocument:

            Version: "2012-10-17"

            Statement: 

              - 

                Effect: Allow

                Action:

                  - "s3:GetObject"

                  - "s3:GetObjectVersion"

                  - "s3:GetBucketVersioning"

                  - "s3:GetObjectAcl"

                  - "s3:PutObject"

                  - "s3:PutObjectAcl"

                  - "s3:PutObjectVersionAcl"                  

                Resource: 

                  - !GetAtt PipelineBucket.Arn

                  - !Join ['', [!GetAtt PipelineBucket.Arn, "/*"]]

              - 

                Effect: Allow  

                Action:

                  - "codebuild:BatchGetBuilds"

                  - "codebuild:StartBuild"

                Resource: "*"

              - 

                Effect: Allow  

                Action:

                  - "codecommit:GetRepository"

                  - "codecommit:GetBranch"

                  - "codecommit:GetCommit"

                  - "codecommit:UploadArchive"

                  - "codecommit:GetUploadArchiveStatus"

                  - "codecommit:CancelUploadArchive"

                Resource: "*"                

      Tags:

        - Key: Name

          Value: !Join ['-', [!Ref AppName, !Ref 'AWS::AccountId', 'PipelineRole', !Ref Stage]]

``` 

After the policies we will create our build project for codebuild to use.

```

  # Create Code Build Project

  CodeBuild:

    Type: 'AWS::CodeBuild::Project'

    Properties:

      Name: !Sub ${AWS::StackName}-CodeBuild

      ServiceRole: !GetAtt CodeBuildRole.Arn

      Artifacts:

        Type: CODEPIPELINE

        Name: MyProject

      Source: 

        Type: CODEPIPELINE

      Environment:

        ComputeType: BUILD_GENERAL1_SMALL

        Type: LINUX_CONTAINER

        Image: aws/codebuild/amazonlinux2-x86_64-standard:3.0

      Source:

        Type: CODEPIPELINE

        # This file (buildspec.yml In Source code) contains commands to Create and Push a docker image to the ECR_REPOSITORY_URI

        BuildSpec: buildspec.yml

      Tags:

        - Key: Name

          Value: !Join ['-', [!Ref AppName, !Ref 'AWS::AccountId', 'BuildProj', !Ref Stage]]

```

We then create our pipeline which has 3 stages source, build and deploy. 

**<u>Note</u>**: For the source declaration, I am using a third party provider which is github. You will need to generate an auth token from github for AWS to use.

```

# Create CodePipeline with 3 stages (Source, Build and Deploy)

  CodePipeline:

    Type: 'AWS::CodePipeline::Pipeline'

    Properties:

      RoleArn: !GetAtt CodePipeLineRole.Arn

      Name: !Join ['-', [!Ref AppName, !Ref 'AWS::AccountId', 'CodePipeLine',!Ref Stage]]

      ArtifactStore:

        Location: !Ref PipelineBucket

        Type: S3

      # Stages declaration

      Stages:

     # Download source code from Github Repo to source-output-artifacts path in S3 Bucket

        - 

          Name: Source

          Actions: 

            - 

              Name: SourceAction

              ActionTypeId: 

                 Category: Source

                 Owner: ThirdParty

                 Provider: GitHub

                 Version: 1

              OutputArtifacts: 

                - 

                  Name: MyApp

              Configuration:                

                 Repo: !Ref GithubRepo

                 Branch: !Ref GithubBranch

                 Owner: !Ref GithubUserName

                 OAuthToken: !Ref GithubOAuthToken

        

        # Build the project using the BuildProject and Output build artifacts to build-output-artifacts path in S3 Bucket

        - 

          Name: Build

          Actions: 

            - 

              Name: BuildAction

              ActionTypeId: 

                Category: Build

                Owner: AWS

                Version: 1

                Provider: CodeBuild

              InputArtifacts: 

                - 

                  Name: MyApp

              OutputArtifacts: 

                - 

                  Name: MyAppBuild

              Configuration:

                ProjectName: !Ref CodeBuild

        # Deploy the project to S3 Bucket for website hosting.

        - 

          Name: Deploy

          Actions: 

            - 

              Name: DeployAction

              ActionTypeId: 

                Category: Deploy

                Owner: AWS

                Version: 1

                Provider: S3

              InputArtifacts: 

                - 

                  Name: MyAppBuild  

              Configuration:                

                BucketName: !Ref DeployBucket

                Extract: 'true'                

      # Create a name tag for the pipeline

      Tags:

        - Key: Name

          Value: !Join ['-', [!Ref AppName, !Ref 'AWS::AccountId', 'CodePipeLine',!Ref Stage]]

```

Now that we have our pipeline, IAM Policies, Build Project, its time to define our S3 Buckets (Store Artifacts, Host Website)

```  

# Create S3 Buckets (Storing Pipeline Artifacts, Website Hosting)

  PipelineBucket: 

    Type: 'AWS::S3::Bucket'

    Properties:

      BucketName: !Join ['-', [!Ref AppName, !Ref 'AWS::AccountId', 'pipelineartifacts', !Ref Stage]]

  DeployBucket:

    Type: 'AWS::S3::Bucket'

    Properties:

      BucketName: !Join ['-', [!Ref AppName, !Ref 'AWS::AccountId', 'website', !Ref Stage]]

      WebsiteConfiguration:

        IndexDocument: index.html

      AccessControl: PublicReadWrite

      CorsConfiguration:

        CorsRules:

        - AllowedOrigins: ['*']

          AllowedMethods: [GET]

  

  # Bucket policy that hosts the website

  DeploymentBucketPolicy: 

    Type: AWS::S3::BucketPolicy

    Properties: 

      Bucket: !Ref DeployBucket

      PolicyDocument: 

        Statement: 

          - 

            Action: 

              - "s3:GetObject"

            Effect: "Allow"

            Resource: 

              Fn::Join: 

                - ""

                - 

                  - "arn:aws:s3:::"

                  - 

                    Ref: DeployBucket

                  - "/*"

            Principal: "*"

         

```

## **<u>Upload CloudFormation Template to AWS</u>**

Great! Now that we have our CloudFormation template created, 

it's now time to upload our template to AWS.

**<u>Step 1</u>**: Log in to your AWS Console and navigate to CloudFormation; you should see a screen similar to this below. Click on create stack.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/30p7soqyohubc00nn0aq.png)

**<u>Step 2</u>**: Choose upload a template file and upload our cloudformation template.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/h57bfvaqgdfkn7hjdesc.png)

**<u>Step 3</u>**: Give the stack a proper name, and give appropriate values to the stack parameters. 

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uwhh7c5h7mveh2gdtda2.png)

**<u>Step 4</u>**: Click next until you're on the last step. Select the checkbox in the last step. This allows AWS to let our template create IAM resources.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/oku9086d9awnq12exmrr.png)

**<u>Step 5</u>**: Once you have created the stack, you can monitor the events of the stack as it is building. Whenever it is fully completed we will get a status of 'CREATE_COMPLETE'

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xwirijx1136qtllg4u46.png)

**<u>Step 6</u>**: Head over to CodePipline and we should see our pipeline in action. 

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/f4k3ov4eiigditjqxf7s.png)

**<u>Step 7</u>**: Navigate over to S3 to view the S3 bucket that is hosting our website. Click on the properties tab and scroll all the way to the bottom. You should see a website url. 

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p8ez0akttygux7r29kof.png)

The URL should load our website and you should see the site below.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/upbye9rnoqxxu4kdth0q.png)

That's it, you have now built your CI/CD Pipeline that automates your website deployment. 

Happy Coding everyone. 👨🏾‍💻


If you aren't satisfied with the build tool and configuration choices, you can `eject` at any time. This command will remove the single build dependency from your project.
