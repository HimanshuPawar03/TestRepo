1. Setting up a 4-stage CI/CD pipeline with manual approvals using AWS CloudFormation:
To create a 4-stage CI/CD pipeline with manual approvals using AWS CloudFormation, let's use 
AWS CodePipeline and AWS CodeDeploy.
-Pipeline stages: 
1>Development(Dev)
2>Quality Assurance(QA)
3>User Acceptance Testing(UAT)
4>Production (Prod).
YAML script to create this pipeline
-->

Resources:
 Pipeline:
  Type: 'AWS::CodePipeline::Pipeline'
   Properties:
    Name: MyPipeline
    RoleArn: !GetAtt PipelineRole.Arn
    Stages:
     - Name: DevEnv
       Actions:
       - Name: SourceCode
         ActionTypeId:
         Category: SourceCode
         Owner: DevEnv
         Version: 'v1.0'
         Provider: GitHub
         Configuration:
          Owner: HimanshuPawar03
          Repo: TestRepo
          Branch: main
          OAuthToken: my-oauth-token
          OutputArtifacts:
          - Name: source-output
       - Name: Build
         ActionTypeId:
         Category: Build
         Owner: AWSUser
         Version: 'v1.0'
         Provider: CodeBuild
        Configuration:
         ProjectName: TestProject
         InputArtifacts:
         - Name: source-output
           OutputArtifacts:
         - Name: build-output
         - Name: Deploy
            ActionTypeId:
             Category: Deploy
             Owner: AWSUser
             Version: 'v1.0'
             Provider: CodeDeploy
           Configuration:
            ApplicationName: CodeDeployApp
            DeploymentGroupName: DevDeployGroup
            InputArtifacts:
            - Name: build-output
#Manual approval to pass to next stage-->
 Blockers:
  - Type: ManualApproval
    Name: DevApproval
    ActionPlan:
    ApproverGroups:
    - Approvers
      CustomData: Dev approval required
    - Name: QA
      Actions:
      - Name: Deploy
        ActionTypeId:
        Category: Deploy
        Owner: AWSUser
        Version: 'v1.0'
        Provider: CodeDeploy
      Configuration:
       ApplicationName: CodeDeployApp
       DeploymentGroupName: QADeployGroup
       InputArtifacts:
       - Name: build-output
# Manual approval required to move to the next stage
 Blockers:
  - Type: ManualApproval
    Name: QAApproval
    ActionPlan:
    ApproverGroups:
     - Approvers
       CustomData: QA approval required
       - Name: UAT
         Actions:
         - Name: Deploy
           ActionTypeId:
           Category: Deploy
           Owner: AWSUser
           Version: 'v1.0'
           Provider: CodeDeploy
           Configuration:
            ApplicationName: CodeDeployApp
            DeploymentGroupName: UATDeployGroup
            InputArtifacts:
            - Name: build-output
#Manual UAT approval required
Blockers:
-Type: ManualApproval
  Name: UATApproval
  ActionPlan:
  ApproverGroups:
  - Approvers
    CustomData: UAT approval required
    - Name: Prod
      Actions:
      - Name: Deploy
        ActionTypeId:
        Category: Deploy
        Owner: AWSUser
        Version: 'v1.0'
        Provider: CodeDeploy
      Configuration:
       ApplicationName: CodeDeployApp
       DeploymentGroupName: ProdDeployGroup
        InputArtifacts:
        - Name: build-output
#Manual prod approval required
Blockers:
- Type: ManualApproval
  Name: ProdApproval
  ActionPlan:
  ApproverGroups:
  - Approvers
    CustomData: Prod approval required
    - Name: Prod
      Actions:
      - Name: Deploy
        ActionTypeId:
        Category: Deploy
        Owner: AWSUser
        Version: 'v1.0'
        Provider: CodeDeploy
      Configuration:
       ApplicationName: CodeDeployApp
       DeploymentGroupName: ProdDeployGroup
        InputArtifacts:
        - Name: build-output
    

2. Terraform Script to Deploy the following:
Infrastructure - (Development Components) Note: Replace Codecommit with Github Batch Job Compoonents 
Code Deployment from Github to ECR
-->

provider "aws" {
 region = "ap-south-1"
}
# Creating IAM role to be used by Batch job
resource "aws_iam_role" "batch_job_role" {
 name = "batch-job-role"
 assume_role_policy = jsonencode({
  Version = "2012-10-17"
  Statement = [
  {
   Action = "sts:AssumeRole"
   Effect = "Allow"
   Principal = {
    Service = "batch.amazonaws.com"
   }
   }
  ]
  })
}
#Attach policy to the Batch job role (policy-> AmazonEC2ContainerRegistryFullAccess)
resource "aws_iam_role_policy_attachment" "batch_job_policy_attachment" {
 policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryFullAccess"
 role = aws_iam_role.batch_job_role.name
}
#Creating Batch job definition
resource "aws_batch_job_definition" "batch_job_definition" {
 name = "regovtest-batch-job-definition"
 type = "container"
 container_properties = jsonencode({
  image = "regov-ecr-repo/regov-image:latest"
  vcpus = 1
  memory = 1024
  command = ["python", "regov_script.py"]
  environment = [
  {
   name  = "ENV_VAR_1"
   value = "regovtest-value1"
   }
{
   name  = "ENV_VAR_2"
   value = "regovtest-value2"
   }
   ]
 })
retry_strategy {
 attempts = 1
 }
 timeout {
  attempt_duration_seconds = 900
  }
  role_arn = aws_iam_role.batch_job_role.arn
}
#Creating ECR repo
resource "aws_ecr_repository" "public.ecr.aws/r5t2e2o1/regov-ecr-repo" {
 name = "regov-ecr-repo"
}
#Creating CodeBuild project to build and push the Docker image to ECR
resource "aws_codebuild_project" "regov_codebuild_proj" {
 name = "regov_codebuild_proj"
 service_role = "arn:aws:iam::role/service-role/codebuild-role"
 artifacts {
  type = "NO_ARTIFACTS"
  }
 environment {
  compute_type = "BUILD_GENERAL1_SMALL"
  image = "aws/codebuild/standard:4.0"
  type = "LINUX_CONTAINER"
  }
 source {
  type = "GITHUB"
  location = "https://github.com/HimanshuPawar03/TestRepo.git"
  git_clone_depth = 1
  buildspec = "testbuildspec.yml"
  }
#Defining build environment variables
environment_variables {
 name  = "ECR_REPO_URI"
 value = public.ecr.aws/r5t2e2o1/regov-ecr-repo
 }
}
#Creating CodePipeline to automatically build and deploy changes from GitHub to ECR
resource "aws_codepipeline" "regov_pipeline" {
 name = "regov-pipeline"
 role_arn = "arn:aws:iam::role/test-pipeline-role"
 artifact_store {
  location = "regov-pipeline-bucket1"
  type = "S3

