# Custom Control Tower Guardrail Example - S3 MFA Delete with Role Exception

AWS Control Tower provides pre-defined guardrails you can optionally enable and enforce on accounts or organizational units (OUs) within your AWS Organization. 

One such guardrail is the ability to block all s3:DeleteObject/DeleteBucket APIs unless the caller has included an MFA token in their request. 

While this may be desirable for human-user use cases, one challenge is that certain AWS services acting on behalf of a user may need to delete objects. In such cases, the IAM role used by the service may not have the ability to provide an MFA token. 

**So, how do we solve this?**

There's a two-step process: 

1. Install the [Customizations for AWS Control Tower](https://aws.amazon.com/solutions/implementations/customizations-for-aws-control-tower/) Solution. This creates a CI/CD pipeline that will deploy your custom guardrails and configurations to Control Tower. 

2. Create your custom guardrails, in the form of CloudFormation templates and/or Service Control Policies (SCPs), along with a manifest file. Zip and upload these files to S3 or push to a Code Commit repository to trigger the pipeline from Step 1. 

**Do I need this solution to add customizations? Could I just deploy my own CloudFormation templates?**

Yes, you could modify SCPs or deploy custom CloudFormation stacks as you like without using the solution above. The benefit of this solution is that the customization pipeline will listen for lifecycle events from Control Tower (such as a new account being added/created) and automatically apply your customizations by re-running the pipeline. This is a convenient time saver and gives you something that you would otherwise need to build yourself. 

## Deployment

1. Clone this repository

2. Deploy the [Customizations for AWS Control Tower](https://aws.amazon.com/solutions/implementations/customizations-for-aws-control-tower/) Solution in your AWS Organization root account in the same region where control tower is set up. 

3. Grant the IAM users or roles permissions to use the KMS key (created by Step 2, above) that is used to encrypt the bucket containing your custom authorizations. This is needed in order to download the sample configuration or upload a new configuration. [Refer to this link for detailed instructions](https://docs.aws.amazon.com/solutions/latest/customizations-for-aws-control-tower/appendix-a.html).

4. The file `custom_config/policies/custom-guardrail-s3mfa-delete.json` contains an SCP that prevents deletes on S3 without MFA, except for an IAM role of your choosing. Open this file and replace the placeholder with a real role ARN, e.g. the role used by your AWS DataSync agent:

    ```json
    "StringNotLike":{
        "aws:PrincipalARN":"arn:aws:iam::111111111111:role/DATASYNC_ROLE_NAME"
    }
    ```

5. the file `custom_config/manifest.yaml` specifies which accounts and/or OUs to which your custom SCP should be applied. Open this file and edit/edit as needed:

    ```yaml
    deployment_targets:
      #accounts:
      #  - 999999999999
      organizational_units:
        - 'Test accounts'
    ```

6. Zip and copy your custom configuration to S3. Within a minute, you should see your pipeline trigger. When the pipeline is complete, your custom SCP should be attached to the accounts and/or OUs you specified: 

    ```sh
    # This should be your AWS Org master account and Control Tower region:
    export AWS_ACCOUNT=999999999999
    export AWS_REGION=us-east-1

    (cd custom_config; zip -r ../test/custom-control-tower-configuration.zip *)

    aws s3 cp  ./test/custom-control-tower-configuration.zip \
    s3://custom-control-tower-configuration-$AWS_ACCOUNT-$AWS_REGION/custom-control-tower-configuration.zip
    ```