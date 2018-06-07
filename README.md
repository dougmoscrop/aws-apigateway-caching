## Summary

It appears that API Gateway does not respect the `requireAuthorizationForCacheControl` setting when overriding method settings.

## Steps to reproduce

1. Deploy the stack:

```bash
aws cloudformation deploy --template-file=cloudformation-template.yml --stack-name=apig-cache-test --region=us-east-1 --capabilities=CAPABILITY_NAMED_IAM
```

2. Export stack outputs to variables:

```bash
export API_ID=$(aws cloudformation --region us-east-1 describe-stacks --stack-name apig-test --query 'Stacks[0].Outputs[?OutputKey==`RestApiId`].OutputValue' --output text) API_ENDPOINT=$(aws cloudformation --region us-east-1 describe-stacks --stack-name apig-test --query 'Stacks[0].Outputs[?OutputKey==`ServiceEndpoint`].OutputValue' --output text)
```

3. Check the stage settings:

```bash
aws apigateway get-stage --rest-api-id=$API_ID --stage-name=dev --region=us-east-1
```

You should see something like this:

```json
{
    "stageName": "dev",
    "description": "dev stage of aws-apig-cache",
    "cacheClusterSize": "0.5",
    "variables": {},
    "cacheClusterEnabled": true,
    "cacheClusterStatus": "CREATE_IN_PROGRESS",
    "deploymentId": "ecimdg",
    "lastUpdatedDate": 1528395583,
    "createdDate": 1528395581,
    "methodSettings": {
        "~1/GET": {
            "throttlingRateLimit": 10000.0,
            "dataTraceEnabled": true,
            "metricsEnabled": false,
            "unauthorizedCacheControlHeaderStrategy": "SUCCEED_WITH_RESPONSE_HEADER",
            "cacheTtlInSeconds": 15,
            "cacheDataEncrypted": false,
            "cachingEnabled": true,
            "throttlingBurstLimit": 5000,
            "requireAuthorizationForCacheControl": true
        }
    }
}
```

4. Wait until the `cacheClusterStatus` is `AVAILABLE`, about 5 minutes. Re-run the command from step 3 to verify.

5. GET the endpoint: `curl $API_ENDPOINT` - This should print a timestamp. If you run the command again within the cache window of 15 seconds, you should see the same timestamp. If you run the command after 15 seconds, the cache TTL should have expired and you see a different timestamp. *This is expected*

6. POST the endpoint: `curl -X POST $API_ENDPOINT` - This should print a timestamp that changes every second you run it, because caching is not enabled for anything but GET. *This is expected*

7. GET the endpoint, but with a cache control header: `curl -H "Cache-control: max-age=0" $API_ENDPOINT` - This should print a timestamp that changes every second you run it, because the Cache-Control header is being respected despite the fact that `requireAuthorizationForCacheControl` is `true`. *This is NOT expected*