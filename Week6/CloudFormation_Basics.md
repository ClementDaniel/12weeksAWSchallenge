# AWS Cloud Formation Foundational Concepts 
AWS CloudFormation is a service provided by Amazon Web Services (AWS) that allows you to define and provision your infrastructure resources in a declarative way. It helps you automate the creation, management, and update of your AWS resources in a consistent and efficient manner. Here are some core concepts of AWS CloudFormation:
## Core Concepts
### Template
A CloudFormation template is a JSON or YAML file that describes the desired state of your AWS resources. It defines the resources, their properties, and any dependencies or relationships between them. Templates can be authored manually or generated using AWS CloudFormation Designer or other tools.
### Stack
A stack is a unit of deployment in CloudFormation. It represents a collection of AWS resources that are created, updated, or deleted together. A stack is created based on a CloudFormation template and has a unique name within an AWS region. Stacks provide isolation and management for the resources within them.
### Resource
A resource is an AWS service component that CloudFormation provisions and manages. Examples of resources include Amazon EC2 instances, Amazon S3 buckets, AWS Lambda functions, and more. Each resource is defined in the CloudFormation template and has properties that specify its configuration.
### Parameter
Parameters allow you to customize the behavior and configuration of your CloudFormation stack. They act as inputs to the template and can be used to provide values that vary across deployments, such as instance sizes, passwords, or environment-specific settings. Parameters can be defined in the template and supplied when creating or updating the stack.
### Output
Outputs provide information about the stack and its resources that can be useful to other parts of your AWS infrastructure or external systems. You can specify the outputs you want to expose in the CloudFormation template, and the values will be available once the stack is created. Outputs can be used for cross-stack references, passing information to other AWS services, or retrieving information after stack creation.
### Stack Policy
A stack policy is a JSON document that defines the permissions for updating or deleting resources within a stack. It allows you to control fine-grained access permissions to protect critical resources. By default, CloudFormation allows all updates and deletions, but a stack policy can be applied to restrict or allow specific actions on resources.
### Change Set
A change set is a summary of proposed changes to a CloudFormation stack. It shows the differences between the current stack configuration and the desired stack configuration defined in a template. Change sets provide a way to review and validate changes before applying them to the stack, helping to prevent unintended modifications.
### Intrinsic functions
Intrinsic functions in AWS CloudFormation are built-in functions that you can use within CloudFormation templates to perform dynamic transformations or retrieve information about your stack and its resources. These functions enable you to manipulate values, perform conditional evaluations, and access metadata during the template processing. Currently, you can use intrinsic functions in resource properties, outputs, metadata attributes, and update policy attributes. 
### Pseudo parameters
Pseudo parameters in AWS CloudFormation are pre-defined variables that you can reference in your CloudFormation templates. These parameters are automatically resolved and populated by AWS CloudFormation during stack creation or update. Pseudo-parameters provide information about the stack, region, and other contextual details.
### Mappings
Mappings in AWS CloudFormation provide a way to define a set of key-value pairs that can be used to customize the resources in your CloudFormation templates based on different conditions, such as regions, environment types, or instance sizes. Mappings allow you to create reusable templates that can be easily adapted to different scenarios without modifying the template structure. You can't include parameters, pseudo parameters, or intrinsic functions in the Mappings section.

These concepts form the foundation of AWS CloudFormation and enable you to define, manage, and automate your infrastructure as code. By using CloudFormation, you can provision and maintain your AWS resources consistently and efficiently, and easily manage changes and updates to your infrastructure over time.

Reference: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html

