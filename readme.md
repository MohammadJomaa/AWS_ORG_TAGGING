# AWS Automated Tagging Strategy - CloudFormation Deployment Guide

## Overview

This solution provides automated resource tagging across your AWS Organization using two CloudFormation templates:

1. **Main Template** (`complete-tagging-strategy.yaml`) - Core tagging infrastructure
2. **Virginia Region Template** (`virginia-region-components.yaml`) - OU tag inheritance components

## Architecture
![Tagging Flow](./Tagging%20Flow.png)
### Main Components (Main Region)
- **Lambda Functions**: Tag resources and evaluate compliance
- **Organization Config Rule**: Monitors resource compliance
- **StackSet**: Deploys cross-account roles to member accounts
- **IAM Roles**: Manages permissions for Lambda functions

### Virginia Region Components (us-east-1)
- **OU Tag Inheritance Lambda**: Propagates tags from OUs to child accounts/OUs
- **EventBridge Rules**: Triggers on OU events (create, move, tag/untag)

## Prerequisites

### AWS Account Setup
- AWS Organizations enabled
- Management account with appropriate permissions
- CloudFormation StackSets enabled
- Config service enabled in target regions

### Required Permissions
Your deployment user/role needs:
- `organizations:*` - Full Organizations access
- `cloudformation:*` - Full CloudFormation access
- `iam:*` - Full IAM access
- `lambda:*` - Full Lambda access
- `config:*` - Full Config access
- `events:*` - Full EventBridge access
- `logs:*` - Full CloudWatch Logs access

### Required Information
Before deployment, gather:
- **Management Account ID**: Your AWS Organizations management account ID
- **Organization ID**: Your AWS Organizations ID (format: `o-xxxxxxxxxx`)
- **Target Regions**: Regions where you want to deploy (e.g., `me-central-1`)

## Template 1: Main Template Deployment

### File: `complete-tagging-strategy.yaml`

**Purpose**: Deploys the core tagging infrastructure in your primary region.

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ManagementAccountId` | String | - | **Required** - Your AWS Organizations management account ID |
| `OrganizationId` | String | - | **Required** - Your AWS Organizations ID (o-xxxxxxxxxx) |
| `ConfigRegion` | String | - | **Required** - Region where Config service is enabled |
| `EnforceValues` | Boolean | false | Whether to enforce tag values (not just keys) |
| `TaggerFunctionName` | String | OrgAccountTagger | Name for the tagging Lambda function |
| `EvaluatorFunctionName` | String | ConfigOrgAccountTagEvaluator | Name for the evaluator Lambda function |
| `MemberRoleName` | String | TagPropagatorRole | Name for the cross-account role in member accounts |
| `DeploymentRegions` | CommaDelimitedList | - | **Required** - Regions to deploy StackSet instances |
| `FailureTolerancePercentage` | Number | 0 | Percentage of accounts that can fail during deployment |
| `MaxConcurrentPercentage` | Number | 10 | Maximum percentage of accounts to process concurrently |

### Resources Created

#### IAM Roles
- **`LambdaTagPropagatorRole`**: Execution role for Lambda functions
  - Organizations permissions
  - Lambda invoke permissions
  - STS assume role permissions
  - Config evaluation permissions

#### Lambda Functions
- **`OrgAccountTaggerFunction`**: Applies missing tags to resources
- **`ConfigOrgAccountTagEvaluatorFunction`**: Evaluates resource compliance

#### Config Components
- **`ConfigInvokePermission`**: Allows Config to invoke evaluator Lambda
- **`OrgAccountTagBaselineRule`**: Organization Config Rule for compliance monitoring

#### StackSet Components
- **`TagPropagatorStackSet`**: Deploys cross-account roles to member accounts
- **`StackSetInstanceCreator`**: Custom resource for automated instance creation
- **`StackSetInstanceCreatorFunction`**: Lambda function for StackSet automation
- **`StackSetInstanceCreatorRole`**: IAM role for StackSet automation

### Deployment Command

```bash
aws cloudformation deploy \
  --template-file complete-tagging-strategy.yaml \
  --stack-name AutomatedTaggingStrategy \
  --parameter-overrides \
    ManagementAccountId=YOUR_MANAGEMENT_ACCOUNT_ID \
    OrganizationId=o-YOUR_ORGANIZATION_ID \
    ConfigRegion=me-central-1 \
    EnforceValues=false \
    TaggerFunctionName=OrgAccountTagger \
    EvaluatorFunctionName=ConfigOrgAccountTagEvaluator \
    MemberRoleName=TagPropagatorRole \
    DeploymentRegions=me-central-1 \
    FailureTolerancePercentage=0 \
    MaxConcurrentPercentage=10 \
  --capabilities CAPABILITY_NAMED_IAM \
  --region me-central-1
```

### Outputs

| Output | Description |
|--------|-------------|
| `LambdaTagPropagatorRoleArn` | ARN of the Lambda execution role |
| `OrgAccountTaggerFunctionArn` | ARN of the tagging Lambda function |
| `ConfigOrgAccountTagEvaluatorFunctionArn` | ARN of the evaluator Lambda function |
| `OrganizationConfigRuleName` | Name of the Organization Config Rule |
| `TagPropagatorStackSetName` | Name of the TagPropagator StackSet |

## Template 2: Virginia Region Template Deployment

### File: `virginia-region-components.yaml`

**Purpose**: Deploys OU tag inheritance components in us-east-1 (required for Organizations events).

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `LambdaTagPropagatorRoleArn` | String | - | **Required** - ARN from main template output |
| `StackName` | String | AutomatedTaggingStrategy | Name of the main stack (for resource naming) |

### Resources Created

#### Lambda Function
- **`InheriteTagOuAccountsFunction`**: Propagates tags from OUs to child accounts/OUs
  - Handles OU tag inheritance
  - Ignores self-triggered events
  - Supports manual invocation

#### EventBridge Rules
- **`CreateAccountTagSyncRule`**: Triggers when new accounts are created
- **`MoveAccountTagSyncRule`**: Triggers when accounts are moved between OUs
- **`TagUnTagAccountTagSyncRule`**: Triggers when OUs are tagged or untagged

#### Permissions
- **`EventBridgeInvokePermission`**: Allows EventBridge to invoke the Lambda function

### Deployment Command

```bash
aws cloudformation deploy \
  --template-file virginia-region-components.yaml \
  --stack-name AutomatedTaggingStrategy-Virginia \
  --parameter-overrides \
    LambdaTagPropagatorRoleArn=arn:aws:iam::YOUR_ACCOUNT_ID:role/LambdaTagPropagatorRole \
    StackName=AutomatedTaggingStrategy \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

### Outputs

| Output | Description |
|--------|-------------|
| `InheriteTagOuAccountsFunctionArn` | ARN of the OU tag inheritance Lambda function |
| `CreateAccountTagSyncRuleName` | Name of the CreateAccount EventBridge rule |
| `MoveAccountTagSyncRuleName` | Name of the MoveAccount EventBridge rule |
| `TagUnTagAccountTagSyncRuleName` | Name of the Tag/Untag OU EventBridge rule |

## Deployment Steps

### Step 1: Deploy Main Template
1. Navigate to your project directory
2. Run the main template deployment command
3. Wait for stack creation to complete
4. Note the outputs, especially `LambdaTagPropagatorRoleArn`

### Step 2: Deploy Virginia Region Template
1. Use the `LambdaTagPropagatorRoleArn` from Step 1
2. Run the Virginia region deployment command
3. Wait for stack creation to complete

### Step 3: Verify Deployment
1. Check CloudFormation stacks in both regions
2. Verify Lambda functions are created
3. Check StackSet instances in member accounts
4. Test the tagging functionality

## Testing the Solution

### Test 1: Manual Lambda Invocation
```bash
# Test the tagger function
aws lambda invoke \
  --function-name OrgAccountTagger \
  --payload '{"accountId": "123456789012"}' \
  response.json

# Test the evaluator function
aws lambda invoke \
  --function-name ConfigOrgAccountTagEvaluator \
  --payload '{"invokingEvent": "{\"configurationItem\":{\"resourceType\":\"AWS::S3::Bucket\",\"resourceId\":\"test-bucket\"}}", "resultToken": "test-token"}' \
  response.json
```

### Test 2: OU Tag Inheritance
```bash
# Test OU tag inheritance manually
aws lambda invoke \
  --function-name InheriteTagOuAccounts \
  --payload '{"ouId": "ou-xxxx-abc", "set": {"Environment": "Production"}}' \
  response.json \
  --region us-east-1
```

### Test 3: Config Rule Evaluation
1. Go to AWS Config console
2. Navigate to Organization Config Rules
3. Check the status of `OrgAccountTagBaselineRule`
4. Review compliance results

## Troubleshooting

### Common Issues

#### 1. StackSet Creation Fails
**Error**: `Resource handler returned message: "Invalid principal in policy"`
**Solution**: Ensure the management account ID is correct and the role exists

#### 2. Lambda Function Errors
**Error**: `Lambda was unable to configure your environment variables`
**Solution**: Check for reserved environment variable names (AWS_REGION, etc.)

#### 3. Cross-Region Deployment Issues
**Error**: `The role defined for the function cannot be assumed by Lambda`
**Solution**: Ensure the role ARN is correct and the role exists in the target region

#### 4. StackSet Instance Creation Fails
**Error**: `Accounts list cannot be empty`
**Solution**: Check Organization ID and ensure accounts exist in the organization

### Debugging Steps

1. **Check CloudFormation Events**
   - Go to CloudFormation console
   - Review stack events for error details

2. **Check Lambda Logs**
   - Go to CloudWatch Logs
   - Review function logs for execution errors

3. **Check StackSet Status**
   - Go to CloudFormation StackSets console
   - Review operation status and error details

4. **Verify Permissions**
   - Ensure deployment user has all required permissions
   - Check IAM policies and trust relationships

## Security Considerations

### IAM Roles
- All roles follow least privilege principle
- Cross-account roles have minimal required permissions
- Lambda execution roles are scoped to specific functions

### Network Security
- Lambda functions run in VPC (if configured)
- No public internet access required
- All communication uses AWS internal networks

### Data Protection
- No sensitive data stored in environment variables
- All API calls use AWS SDK with proper authentication
- CloudTrail logging enabled for audit trails

## Maintenance

### Regular Tasks
1. **Monitor Compliance**: Check Config rule compliance regularly
2. **Review Logs**: Monitor Lambda function logs for errors
3. **Update Dependencies**: Keep Lambda runtime versions current
4. **Test Functionality**: Periodically test tagging and inheritance

### Updates
1. **Template Updates**: Use CloudFormation change sets for updates
2. **Code Updates**: Update Lambda function code as needed
3. **Parameter Changes**: Modify parameters through CloudFormation console

## Cost Optimization

### Lambda Functions
- Functions use minimal memory allocation
- Timeout values are optimized for typical workloads
- Consider reserved concurrency for high-volume scenarios

### CloudWatch Logs
- Set log retention policies to avoid excessive costs
- Monitor log group sizes and adjust retention as needed

### Config Service
- Monitor Config rule evaluation costs
- Consider evaluation frequency for cost optimization

## Support and Documentation

### Additional Resources
- [AWS Organizations Documentation](https://docs.aws.amazon.com/organizations/)
- [AWS Config Documentation](https://docs.aws.amazon.com/config/)
- [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/)
- [CloudFormation StackSets Documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html)

### Getting Help
1. Check AWS documentation for specific service issues
2. Review CloudFormation and Lambda logs for error details
3. Test individual components to isolate issues
4. Consider AWS Support for complex organizational issues

---

**Note**: This solution is designed for AWS Organizations with multiple accounts. Ensure you have proper permissions and understand the implications of deploying across your organization before proceeding.
