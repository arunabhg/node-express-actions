# Node.js Express CI to AWS S3 artifact using GitHub Actions OIDC - starter template

<br />

***TLDR*:*** Using the below steps, whenever we make any changes to the code, it is automatically pushed to S3 with the help of OIDC (Open ID Connect) using GitHub Action's CI pipeline. We don't using any Secrets in GitHub Actions. Using OIDC, the connection between AWS and GitHub Actions exists for a short duration (min 1 hr) which can be extended.

### Steps

1.  Create a new repo or clone this repo.
    <br /><br />
2.  We don't need to create a GitHub Secret or an IAM User to upload artifacts to S3 bucket. All we need is to create a role with the specific access. _Refer_ - [AWS-Actions - Configure AWS Credentials](https://github.com/aws-actions/configure-aws-credentials)<br /><br />
3.  Using **OpenID Connect (OIDC)** we can adopt the following practices:<br />
    3.1 No need to configure long-lived GitHub cloud secrets.<br />
    3.2 We have more granular control over how workflows can use credentials.<br />
    3.3 Using OIDC, the cloud provider (AWS, Azure, GCP), issues a short-lived access-token that is only valid for a single job, and then automatically expires. <br />
    _Refer_ - [Overview of OpenId Connect](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)<br /><br />
4.  Go to AWS console -> IAM -> Identity Providers & click on **Add Provider**. <br /><br />
5.  Select OpenID Connect and for the provider URL: `https://token.actions.githubusercontent.com`<br /><br />
6.  Click on **Get Thumbprint**.<br /><br />
7.  For the audience, use: `sts.amazonaws.com` and add the provider.<br /><br />
8.  Click on Assign Role -> Create a new role. <br />
    8.1 AWS creates an identity type of **Web Identity** with audience of `sts.amazonaws.com`. Click **Next: Permissions**<br />
    8.2 You can give S3FullAccess. **_Note -_** Permissions (List, Read, Write) should be for the specific buckets, not full access in production.<br />
    8.3 Give the role a name, eg., _Github-actions-role_, add description, and click on **Create Role**.<br /><br />
9.  Go to the trust relationship of the role and click Edit Trust Policy.<br />
    Add the line to it: `"token.actions.githubusercontent.com:sub": "repo:octo-org/octo-repo:ref:refs/heads/octo-branch"`<br />
    Change the repo owner, name & branch as it applies & Click on **Update Policy**.<br /> For eg., in our case, the string will be - `"token.actions.githubusercontent.com:sub": "repo:arunabhg/node-express-actions:ref:refs/heads/main"`<br />
    **_Note-_** In some cases we may get error `Error: Not authorized to perform sts:AssumeRoleWithWebIdentity`. In That case we have to Edit the Policy & write the policy as:<br />

    ```
    "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:arunabhg/node-express-actions:*"
                }
            }

    ```

_Here the \* after the repo name means the repo can be accessed by all refs and not only main._

10. Go to the YAML file in your project's GitHub Workflows.<br />
    10.1 Add an env section with the bucket name, region & permissions.<br />
    `env:
BUCKET_NAME: "<aws-bucket-name>"
AWS_REGION: "<aws-region>"
GITHUB_REF: "main"`<br>
    10.2 Add a permissions section in the jobs part which lists the read and write permissions: <br />
    `  permissions:
    id-token: write
    contents: read #required for actions/checkout` <br />
    Put this before the steps part. <br />
    10.3 Add a section to configure the AWS credentials as the first step - <br />

     ```
        - name: Configure AWS Credentials
            uses: aws-actions/configure-aws-credentials@v1
            with:
              role-to-assume: arn:aws:iam::754345194439:role/GitHub-actions-role
              role-session-name: Github-Actions-Role
              aws-region: ${{ env.AWS_REGION }}
    ```

    _Note -_ Remember to change the arn and role-session-name as specified in the IAM Role, otherwise the Action will fail.<br />
    10.4 Add a step to create a SHA hash. Add this section just below the above one.<br />

    ```
        - name: Extract branch name
        shell: bash
        run: echo "##[set-output name=branch;]$(echo ${GITHUB_REF#refs/heads/})"
        id: extract_branch
        - name: Extract commit hash
        shell: bash
        run: echo "##[set-output name=commit_hash;]$(echo "$GITHUB_SHA")"
        id: extract_hash
    ```

    10.5 As a last step in the workflow, make a directory in S3 and upload the artifact as a zip file to the folder in S3.
    Add the last step at the end of the workflow - <br />

    ```
        - name: Make Artifact directory
        run: mkdir -p ./artifacts

        # Copy build directory to S3
        - name: Copy build to S3
        run: |
            zip -r ./artifacts/project.zip . -x node_modules/**\* .git/**\* dist/**\* dist/**\*
            aws s3 sync  ${GITHUB_WORKSPACE}/artifacts s3://${{ env.BUCKET_NAME }}/${{ steps.extract_branch.outputs.branch }}/latest
    ```

---

***Note -*** Remember to give the correct names to your S3 bucket, folder, session, role-session, aws-region, etc. in the workflow, otherwise the workflow will fail.
