# turn-server-deploy
Deploy a turn server to AWS with CloudFormation

We have developed a script that can automatically deploy a TURN server to Amazon Web Services (AWS). You can follow the instructions [here](https://github.com/freddyaboulton/turn-server-deploy) or this guide.

### Prerequisites

Clone the following [repository](https://github.com/freddyaboulton/turn-server-deploy) and install the `aws` cli if you have not done so already (`pip install awscli`).

Log into your AWS account and create an IAM user with the following permissions:

- [AWSCloudFormationFullAccess](https://us-east-1.console.aws.amazon.com/iam/home?region=us-east-1#/policies/details/arn%3Aaws%3Aiam%3A%3Aaws%3Apolicy%2FAWSCloudFormationFullAccess)
- [AmazonEC2FullAccess](https://us-east-1.console.aws.amazon.com/iam/home?region=us-east-1#/policies/details/arn%3Aaws%3Aiam%3A%3Aaws%3Apolicy%2FAmazonEC2FullAccess)


Create a key pair for this user and write down the "access key" and "secret access key". Then log into the aws cli with these credentials (`aws configure`).

Finally, create an ec2 keypair (replace `your-key-name` with the name you want to give it).

```
aws ec2 create-key-pair --key-name your-key-name --query 'KeyMaterial' --output text > your-key-name.pem
```

### Deployment

Open the `parameters.json` file and fill in the correct values for all the parameters:

- `KeyName`: The key file we just created
- `TurnUserName`: The username needed to connect to the server
- `TurnPassword`: The password needed to connect to the server
- `InstanceType`: One of the following values `t3.micro`, `t3.small`, `t3.medium`, `c4.large`, `c5.large`


Then run the deployment script:

```bash
aws cloudformation create-stack \
  --stack-name turn-server \
  --template-body file://deployment.yml \
  --parameters file://parameters.json \
  --capabilities CAPABILITY_IAM
```

You can then wait for the stack to come up with:

```bash
aws cloudformation wait stack-create-complete \
  --stack-name turn-server
```

Next, grab your EC2 server's public ip with:

```
aws cloudformation describe-stacks \
  --stack-name turn-server \
  --query 'Stacks[0].Outputs' > server-info.json
```

The `server-info.json` file will have the server's public IP and public DNS:

```json
[
    {
        "OutputKey": "PublicIP",
        "OutputValue": "35.173.254.80",
        "Description": "Public IP address of the TURN server"
    },
    {
        "OutputKey": "PublicDNS",
        "OutputValue": "ec2-35-173-254-80.compute-1.amazonaws.com",
        "Description": "Public DNS name of the TURN server"
    }
]
```

Finally, you can connect to your EC2 server from the gradio WebRTC component via the `rtc_configuration` argument:

```python
import gradio as gr
from gradio_webrtc import WebRTC
rtc_configuration = {
    "iceServers": [
        {
            "urls": "turn:35.173.254.80:80",
            "username": "<my-username>",
            "credential": "<my-password>"
        },
    ]
}

with gr.Blocks() as demo:
    webrtc = WebRTC(rtc_configuration=rtc_configuration)
```
