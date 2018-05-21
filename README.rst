TL;DR summary
-------------

This is TL;DR summary to just get the node up. It assumes that you have an AWS account set up
with programmatic access.

1. Open terminal. Install ``aws`` and ``jq`` if not installed.


2. Setup which account and region you work with.

   ::

      export AWS_DEFAULT_REGION=eu-central-1
      # matching entry in ~/.aws/credentials
      export AWS_DEFAULT_PROFILE=...

3. Create repository, build image and push it.

   ::

      bash build_and_upload.sh

   This command creates an ECR repository under your account and pushes there a slightly
   customized image of Parity client.


4. Specify parameters of the node.

   ::

      cd cloudformation
      cp stack-parameters.default.json stack-parameters.json
      $EDITOR stack-parameters.json

   In here you need to specify:
     - ``VpcId`` to run the chain
     - ``DNSName`` to register for your node (eg. ``mainnet.rumblefishdev.com``)


5. Create CloudFormation stack.

   ::

      bash -x create-stack
