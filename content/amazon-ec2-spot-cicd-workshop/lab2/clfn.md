+++
title = "Code snippet: The Test Environment CloudFormation template"
weight = 10
+++
Below is a sanitized version of the CloudFormation template that will be used to launch your test environments using an Amazon EC2 Spot Fleet provisioned from an EC2 Launch Template:
```yaml
AWSTemplateFormatVersion: '2010-09-09'

Description: A CloudFormation template that will deploy a test environment as a part of the Amazon EC2 Spot CI/CD Workshop. This template is provided as-is under a modified MIT license - please see https://github.com/aws-samples/amazon-ec2-spot-cicd-workshop/blob/master/LICENSE

Parameters:
  KeyPair:
    Description: The Key Pair created in Step 3 of the Preparation Lab
    Type: AWS::EC2::KeyPair::KeyName

  CurrentIP:
    Description: The IP address supplied by the workshop participant when provisioning the Master CloudFormation template for this workshop.
    Type: String

  AMILookupLambdaFunctionARN:
    Description: The ARN correspoding to the AMI Lookup Lambda Function created by the Master CloudFormation template.
    Type: String

  DeploymentArtifactsS3Bucket:
    Description: The S3 bucket created by the Master CloudFormation template to host deployment artifacts.
    Type: String

  VPC:
    Description: The VPC created by the Master CloudFormation template, where the test environment will be provisioned.
    Type: String

  SubnetA:
    Description: Subnet A created by the Master CloudFormation template, one of three which will be used by the test environment.
    Type: String
    
  SubnetB:
    Description: Subnet B created by the Master CloudFormation template, one of three which will be used by the test environment.
    Type: String
    
  SubnetC:
    Description: Subnet C created by the Master CloudFormation template, one of three which will be used by the test environment.
    Type: String
    

Resources:

  IAMRoleGameOfLife:
    Type: AWS::IAM::Role
    # DependsOn: None
    # DependedOn: InstanceProfileGameOfLife
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
        - Effect: Allow
          Principal:
            Service:
            - ec2.amazonaws.com
          Action:
          - sts:AssumeRole
      Path: /
      Policies:
      - PolicyName: Test
        PolicyDocument:
          Version: 2012-10-17
          Statement:
          - Effect: Allow
            Action: s3:GetObject
            Resource: !Sub "arn:aws:s3:::${DeploymentArtifactsS3Bucket}/*"

  InstanceProfileGameOfLife:
    Type: AWS::IAM::InstanceProfile
    DependsOn: IAMRoleGameOfLife
    # DependedOn: GameOfLifeLaunchTemplate
    Properties:
      Path: '/'
      Roles:
      - !Ref IAMRoleGameOfLife

  AMILookupCustomResource: # A custom resource that provides the latest Amazon Linux AMI via AMILookupCustomResource.Id
    Type: Custom::AMILookup
    # DependsOn: 
    # DependedOn:
    Properties:
      Architecture: HVM64
      Region: !Ref AWS::Region
      ServiceToken: !Ref AMILookupLambdaFunctionARN

  SecurityGroupGameOfLifeALB: # A Security Group that allows ingress access for HTTP on ALBs and used to access the Game of Life test environment
    Type: AWS::EC2::SecurityGroup
    # DependsOn: None
    # DependedOn: None
    Properties:
      GroupName: Spot CICD Workshop Game of Life ALB Security Group
      GroupDescription: A Security Group that allows ingress access for HTTP on ALBs and used to access the Game of Life test environment
      SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 80
        ToPort: 80
        CidrIp: 0.0.0.0/0
      VpcId: !Ref VPC

  SecurityGroupGameOfLifeEC2: # A Security Group that allows ingress access for SSH and the default port that the Game of Life test application will run on
    Type: AWS::EC2::SecurityGroup
    DependsOn: SecurityGroupGameOfLifeALB
    # DependedOn: JenkinsOnDemandEC2Instance
    Properties:
      GroupName: Spot CICD Workshop Game of Life EC2 Security Group
      GroupDescription: A Security Group that allows ingress access for SSH and the default port that a Jenkins Master will run on
      SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 22
        ToPort: 22
        CidrIp: !Ref CurrentIP
      - IpProtocol: tcp
        FromPort: 8080
        ToPort: 8080
        CidrIp: !Ref CurrentIP
      - IpProtocol: tcp
        FromPort: 8080
        ToPort: 8080
        SourceSecurityGroupId: !Ref SecurityGroupGameOfLifeALB
      VpcId: !Ref VPC

  GameOfLifeALB: # This is the Application Load Balancer that resides in front of the Game of Life test environment and is responsible for port-mapping requests from TCP:80 to TCP:8080
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    DependsOn: SecurityGroupGameOfLifeALB
    # DependedOn: GameOfLifeALBListener
    Properties:
      Name: GameOfLifeALB
      Scheme: internet-facing
      SecurityGroups:
      - !Ref SecurityGroupGameOfLifeALB
      Subnets:
      - !Ref SubnetA
      - !Ref SubnetB
      - !Ref SubnetC

  GameOfLifeALBTargetGroup: # This is the Target Group used by the GameOfLifeALB load balancer
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    # DependsOn: None
    # DependedOn: GameOfLifeALBListener, GameOfLifeALBListenerRule
    Properties:
      HealthCheckIntervalSeconds: 15
      HealthCheckPath: /gameoflife/
      HealthCheckPort: 8080
      HealthCheckProtocol: HTTP
      HealthCheckTimeoutSeconds: 5
      HealthyThresholdCount: 2
      Matcher:
        HttpCode: 200
      Name: GameOfLifeALBTargetGroup
      Port: 8080
      Protocol: HTTP
      UnhealthyThresholdCount: 4
      VpcId: !Ref VPC

  GameOfLifeALBListener: # This is the ALB Listener used to access the Game of Life test environment
    Type: AWS::ElasticLoadBalancingV2::Listener
    DependsOn: 
    - GameOfLifeALB 
    - GameOfLifeALBTargetGroup
    # DepenededOn: None
    Properties:
      DefaultActions:
      - Type: forward
        TargetGroupArn: !Ref GameOfLifeALBTargetGroup
      LoadBalancerArn: !Ref GameOfLifeALB
      Port: 80
      Protocol: HTTP

  GameOfLifeALBListenerRule: # The ALB Listener rule that forwards all traffic destined for the Game of Life test environment to the appropriate Target Group
    Type: AWS::ElasticLoadBalancingV2::ListenerRule
    DependsOn: 
    - GameOfLifeALBListener
    - GameOfLifeALBTargetGroup
    # DependedOn: None
    Properties:
      Actions:
      - Type: forward
        TargetGroupArn: !Ref GameOfLifeALBTargetGroup
      Conditions:
      - Field: path-pattern
        Values:
        - "/*"
      ListenerArn: !Ref GameOfLifeALBListener
      Priority: 1

  GameOfLifeLaunchTemplate: # The Launch Template that will be used to deploy the test environment
    Type: AWS::EC2::LaunchTemplate
    DependsOn: SecurityGroupGameOfLifeEC2
    # DependedOn: None
    Properties:
      LaunchTemplateName: GameOfLifeLaunchTemplate
      LaunchTemplateData:
        BlockDeviceMappings:
        - DeviceName: "/dev/xvda"
          Ebs:
            DeleteOnTermination: 'true'
            VolumeSize: 8
            VolumeType: gp2
        IamInstanceProfile: 
          Name: !Ref InstanceProfileGameOfLife
        ImageId: !GetAtt AMILookupCustomResource.Id
        InstanceType: t3.micro
        KeyName: !Ref KeyPair
        SecurityGroupIds:
        - !Ref SecurityGroupGameOfLifeEC2
        TagSpecifications:
        - ResourceType: instance
          Tags:
          - Key: Name
            Value: Game of Life Test Instance
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash
            # Install all pending updates to the system
            yum -y update
            # Install Java 8 & Apache Tomcat 8
            yum -y install java-1.8.0-openjdk tomcat8
            # Download the Game of Life build artifact to Tomcat's webapps directory
            aws s3 cp s3://${DeploymentArtifactsS3Bucket}/gameoflife-web/target/gameoflife.war /usr/share/tomcat8/webapps/gameoflife.war
            # Start the Tomcat8 Service
            service tomcat8 start

  GameOfLifeSpotFleet: # The Spot Fleet definition that will launch the EC2 instances that will comprise the test environment
    Type: AWS::EC2::SpotFleet
    DependsOn: GameOfLifeLaunchTemplate
    # DependedOn: None
    Properties:
      SpotFleetRequestConfigData:
        AllocationStrategy: diversified
        IamFleetRole: !Sub arn:aws:iam::${AWS::AccountId}:role/aws-ec2-spot-fleet-tagging-role
        LaunchTemplateConfigs:
        - LaunchTemplateSpecification: 
            LaunchTemplateName: GameOfLifeLaunchTemplate
            Version: !GetAtt GameOfLifeLaunchTemplate.LatestVersionNumber
          Overrides:
          - SubnetId: !Ref SubnetA
        - LaunchTemplateSpecification: 
            LaunchTemplateName: GameOfLifeLaunchTemplate
            Version: !GetAtt GameOfLifeLaunchTemplate.LatestVersionNumber
          Overrides:
          - SubnetId: !Ref SubnetB
        - LaunchTemplateSpecification: 
            LaunchTemplateName: GameOfLifeLaunchTemplate
            Version: !GetAtt GameOfLifeLaunchTemplate.LatestVersionNumber
          Overrides:
          - SubnetId: !Ref SubnetC
        LoadBalancersConfig:
         TargetGroupsConfig:
            TargetGroups:
            - Arn: !Ref GameOfLifeALBTargetGroup
        ReplaceUnhealthyInstances: true
        TargetCapacity: 2
        Type: maintain

Outputs:
  GameOfLifeDNSName:
    Description: The DNS name of the Application Load Balancer that is used to gain access to the Game of Life testing environment.
    Value: !GetAtt GameOfLifeALB.DNSName
```