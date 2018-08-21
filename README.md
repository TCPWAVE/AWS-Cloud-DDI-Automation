# 1. Cloud-DDI Automation
The purpose of this content is to give a high-level overview of configuring AWS Cloud Ecosystem to automatically register and deregister EC2 Instance information into TCPWave IPAM and private DNS and to ftp the CSV of the consolidated state changes that have taken place in the specified number of hours till the current time.
# 2. Architecture Overview

![architecture-diagram](https://user-images.githubusercontent.com/4006576/41895756-78bff826-7940-11e8-9ea4-df6e7ca80c5b.png)

Architectural diagram provided above depicts the flow of events happening from AWS EC2 system to TCPWave IPAM and External Server using TCPWave DDI automation solution. Below is the description of the events.
1.	AWS EC2 instance state change invokes preconfigured Lambda function through AWS CloudWatch event listener.
2.	Lambda function invokes TCPWave IPAM’s Rest API though SSL authentication with the required instance details it gets from aws-sdk EC2 API, to create or delete object in TCPWave IPAM.
3.	TCPWave IPAM sends DDNS updates to internal DNS appliances and Cloud DNS eco system.
4.	DDI Automation solution provides another Lambda function to configure to CloudWatch event which sends consolidated CSV of the events that have taken place on AWS EC2 instances in the last 24 hours, to external server through FTP. Sample CSV is given below.

| Add/Delete | IP | Name | TimeStamp |
|------------|----|------|-----------|
| Delete | 172.31.0.183 | TEST-LAMBDA | (Wed Jun 06 2018 07:09:52 GMT+0000 (UTC)) |
| Add | 172.31.0.164 | jahn-linux | (Wed Jun 06 2018 07:37:32 GMT+0000 (UTC)) |
| Delete | 172.31.0.33 | ip-172-31-0-33 | (Wed Jun 06 2018 08:44:22 GMT+0000 (UTC)) |

The CloudWatch Event Listener set up by the user tracks the instance state changes – such as running, stopped and terminated and invokes defined AWS Lambda Function. This function gets invoked whenever an instance state is changed. The function then communicates with TCPWave IPAM using REST API to update the object information. When the state of an EC2 instance changes to “running”, the EC2 instance object is added to the TCPWave IPAM. When an EC2 instance is stopped or terminated, the EC2 instance object information gets deleted from the TCPWave IPAM via the AWS Lambda function. Similarly other Lambda function mentioned in the event 4 sends CSV file to external server.

# 3. Configuration Steps to send updates to TCPWave IPAM
## 3.1 Prepare zip file to upload to Lambda
### Steps to create and import certificates
In this step the user needs to create appliance and client certificates to import into IPAM and to use them to communicate with IPAM Rest API from AWS Lambda function.
To create certificates on a Linux system, follow the below steps
1. Create a rootCA.key and a self-signed certificate and sign with the rootCA.key
     1.  openssl genrsa -des3 -out rootCA.key 4096
     2. openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.crt

2. Create client.key and client.crt by following below steps
     1. openssl genrsa -out client.key 2048
     2. openssl req -new -key client.key -out client.csr
     3. openssl x509 -req -in client.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out client.crt -days 500 -sha256

### Steps to import Certificates to IPAM
1.  Import the rootCA certificate to the IPAM trust store: Use the Appliance certificates GUI (Administration -> Security Management-> Appliance Certificates) and set the Trust CA checkbox. Select rootCA.crt file that was generated earlier and the rootCA.key file. Enter the password used for the private key generation in Private Key Password field. Certificate storage password is the TIMS keystore password. TCPWave provides a default keystore password which can be changed from “Change Store Password” option in the Appliance Certificates page.
2.  Import the client certificate in the User Certificates (Administration -> Security Management-> User Certificates) screen. Here, select client.crt file and the associated admin from the dropdown. Click on OK.
### Steps to create zip file
Create a zip file consisting of below files
1.	client.crt
2.	client.key
3.	index.js

Where client.crt and client.key files are the files created in the step “Steps to create and import certificates”.

The content of index.js file is as below. The variables highlighted in yellow have to be updated according to the user’s environment. 

    exports.handler = (event, context, callback) => {
    var orgName = "Internal";
    var domainName = "example.com";
    var hostIp = "20.3.4.5”;
    var ttl = "1200";
    var objectType = "AWS Instance";
    var AWS = require('aws-sdk');
    var cData = JSON.stringify(event, null, 2);
    var arr = [];
    if(cData)
    {
      arr = cData.split(" ");
    }
    console.log('Received event:', cData); 
    var instanceId = arr[0].substring(1);
    var status = arr[1].substring(0, arr[1].length-1);
    console.log("Status:"+status);
    var ec2 = new AWS.EC2();
    var params = { InstanceIds: [instanceId]};
    var name;
    var ip;
    ec2.describeInstances(params, function(err, data) {
    if (err) 
    {
      console.log(err, err.stack); 
    }// an error occurred
    else   
    {
      var res = data.Reservations;
      for(var i =0; i<res.length; i++)
      {
        var instances = res[i].Instances;
         console.log("Instances:"+instances);
        for(var j =0; i<instances.length; i++)
        {
          console.log("Instance:"+i);
          var instance = instances[j];
          ip = instance.PrivateIpAddress;
          var tags = instance.Tags;
          
          for(var k =0; k<tags.length; k++)
          {
            var tag = tags[k];
            if(tag.Key == "TWC Organization")
            {
              orgName = tag.Value;
            }
            else if(tag.Key == "TWC Domain")
            {
              domainName = tag.Value;
            }
            else if(tag.Key == "TWC DNS Name")
            {
              name = tag.Value;
            }
          }
          console.log("Organization:"+orgName);
          console.log("Domain:"+domainName);
          if(instance)
          {
            break;
          }
        }
        if(instances)
        {
          break;
        }
      }
      console.log("Instance:"+name+" "+ip);  
      var ipArr = ip.split(".");
      var path;
    if(status == 'running')
    {
      path = '/tims/rest/object/add';
    }
    else
    {
      path = '/tims/rest/object/delete-multiple';
    }
    var reqData = {};
    
    if(status == 'running')
    {
      reqData.addr1 = ipArr[0];
      reqData.addr2 = ipArr[1];
      reqData.addr3 = ipArr[2];
      reqData.addr4 = ipArr[3];
      reqData.class_code = objectType;
      reqData.domain_name = domainName;
      reqData.ttl = ttl;
      if(name == undefined || name == "")
      {
        name = "ip-"+ipArr[0]+"-"+ipArr[1]+"-"+ipArr[2]+"-"+ipArr[3];
      }
      reqData.name = name;
      reqData.alloc_type = "1";
      reqData.organization_name = orgName;
      reqData.dyn_update_rrs_a = true;
      reqData.dyn_update_rrs_cname = true;
      reqData.dyn_update_rrs_mx = true;
      reqData.dyn_update_rrs_ptr = true;
      reqData.update_ns_a = true;
      reqData.update_ns_ptr = true;
    }
    else
    {
      var addrArr = [ip];
      reqData.addressArray = addrArr;
      reqData.organization_name = orgName;
    }
     console.log("Event::"+event);
    
    var https = require('https');
      const tls = require('tls');
      var fs = require('fs');
      var options = {
        host: hostIp,
        port: '7443',
        path: path,
        method: 'POST',
        key: fs.readFileSync('client.key'),
        cert: fs.readFileSync('client.crt')
       };
    process.env['NODE_TLS_REJECT_UNAUTHORIZED'] = '0';
    var postHeaders = {'Content-Type':'application/json'};
      var jsonObject;
      if ( reqData )
      {
        jsonObject = JSON.stringify(reqData);
      }
      if ( jsonObject )
      {
        console.log(' ');
        console.log(jsonObject);
        postHeaders["Content-Length"] = Buffer.byteLength(jsonObject, 'utf8');
      }
      options.headers = postHeaders;
      console.log("Headers:"+postHeaders);
      var reqPost = https.request(options, function(res){
      console.log("Status Code: ", res.statusCode);
        res.on('data', function(d){
          process.stdout.write(d);
        });
      });
    if ( jsonObject )
      reqPost.write(jsonObject);
    else
      reqPost.write("{}");
    
      reqPost.on('error', function(e){
        console.error('Error: '+e);
         });
         }// successful response
       }); 
            callback(null, 'EC2InstanceStateChange execution completed.');
        };
### Variables:
Below is the information about the variables highlighted in the above function.

| Variable | Example Value | Description |
|----------|---------------|-------------|
| orgName |	Internal |	Organization Name in which the subnet presents in the TCPWave IPAM. |
| domainName |	example.com |	The domain to which object belongs in TCPWave IPAM. |
| hostIp |	20.3.4.5 |	IP Address of the TCPWave IPAM. |
| ttl |	1200 | Time To Live value. |

### Custom tags:
**Important**: The above function provides flexibility for the user to define Organization, Domain and DNS Name of the object as custom tags at the time of creation of EC2 instance with names ‘TWC Organization’, ‘TWC Domain’ and ‘TWC DNS Name’ respectively. 

If ‘TWC Organization’ tag for the EC2 instance is not available, orgName variable value will be taken as organization input to create object in IPAM.

If ‘TWC Domain tag’ for the EC2 instance is not available, domainName variable value will be taken as domain name input to create object in IPAM.

If ‘TWC DNS Name’ tag for the EC2 instance is not available, place holder name formed by appending IP Address value to ‘ip’ (example: ip-10-2-3-4) will be taken as object name input to create object in IPAM.

## 3.2 Lambda and CloudWatch Configuration
Create a Lambda Function from the AWS Lambda page.
1.	Click on “Create function” button
2.	Select Node.js 8.10 under Runtime
3.	Under Role, select “Create a custom role” option
4.	Save the custom role “lambda_execution” by clicking on “Allow” button
![lambda](https://user-images.githubusercontent.com/4006576/41895928-dc273c3a-7940-11e8-8717-0cde92dd35ce.png)
5.	Click on “Create function” 
6.	Under Designer, select Add triggers > CloudWatch Events
7.	Click on the “Configuration required” link under the CloudWatch Events trigger to configure the triggers
8.	Scroll down to Configure triggers section and select “Create a new rule”
9.	Enter new rule name and select “Event pattern” as Rule type
10.	Under the first dropdown select “EC2”
11.	Under the second dropdown select “EC2 instance state-change notification
12.	Check the “Detail” checkbox 
13.	Check the “State” checkbox
14.	Under State, select running, shutting-down and stopped options
15.	As a result, the Event pattern preview should look as below:

        {
            "source": [
            "aws.ec2"
          ],
          "detail-type": [
            "EC2 Instance State-change Notification"
          ],
          "detail": {
            "state": [
              "running",
              "shutting-down",
              "stopped"
            ]
          }
        }
16.	Check “Enable trigger”
17.	Click on Add button
18.	Click on “Save” button on the top of the Lambda Function page to save the changes	
19.	Go back to the Functions page
20.	Click on the newly created Function
21.	In Basic Settings, update the Timeout value to 1 min to avoid the function timeout issues due to network latency.
22.	Under the Functions code section, select ‘Upload a .ZIP file’ option for Code entry type. 
23.	Upload the zip file created in the step ‘Steps to create zip file’.
24.	Save the changes by clicking on the Save button on the top right corner of the page.
25.	Updating role ‘lambda_execution’

        1.	Go to Roles page in IAM.
        
        2.	Open the role ‘lambda_execution’ created in step 4.
        
        3.	Attach policies: AmazonEC2ReadOnlyAccess and AWSLambdaVPCAccessExecutionRole to the role. The role should look as below.
 ![lambda1](https://user-images.githubusercontent.com/4006576/41896082-50d1363a-7941-11e8-90e0-b1bb8b2049df.png)
        
26.	Updating CloudWatch Rule.

        1.	Go to Rules page of CloudWatch and open the rule created under Designer section of Lambda function.
        
        2.	Click on Edit link under the Actions dropdown on the top right corner of the page.
        
        3.	Under Targets -> Lambda Function -> Configure Input , select Input Transformer radio button.
        
        4.	Paste below content in the first input box.
        {"instance":"$.detail.instance-id","state":"$.detail.state"}
        
        5.	Paste below content in the second input box (along with the double quotes).
        "<instance> <state>"
        
        6.	Click on Configure Details button on the bottom right corner of the page.
        
CloudWatch rule should look as below.

![lambda2](https://user-images.githubusercontent.com/4006576/41896091-553ff1b6-7941-11e8-90ce-657c41798bde.png)

Lambda function should look as below.

![lambda3](https://user-images.githubusercontent.com/4006576/41896108-5bb0572a-7941-11e8-96d0-ba50e0d3d6ae.png)
        
After the above configuration is completed successfully, Rest API on TCPWave IPAM will be invoked on EC2 instance state change.  Object will be created in the IPAM when the instance state changes to Running. And object will be deleted from IPAM when the instance is Terminated or Stopped.

# 4. Troubleshooting Steps

1.	Check the logs in CloudWatch -> Logs -> Logs Group 
2.	If Object is not created/deleted in TCPWave IPAM, check the following.

        a.	Adjust the Timeout value in the Lambda function.

        b.	Check /opt/tcpwave/logs/Tims.log in IPAM for more information.

        c.	Check if the certificates are matching in IPAM and Lambda function.

        d.	orgName and domainName are matching with IPAM.

        e.	hostIp variable is properly set up.

        f.	Custom tags on EC2 instance are matching with IPAM.
  
3.	Contact TCPWave Customer Care for further assistance.

# 5. Configuration Steps to send CSV to external server
**Steps to create a Lambda function**
1.	Create Lambda function by uploading the send_csv_to_external_server_fn.zip file. Update the ftp parameters and the file path in index.js according to the ftp server details. Update the value of the parameter ‘rotationDuration’ as per the requirement. Default value is 24 hours.
Please refer to ‘Variables’ and ‘Custom Tags’ sections provided above to know more information about other variables.
2.	Increase the timeout value to avoid timeout issue because of network latency.
3.	Assign execution role to the function which has below policies attached to it. AmazonEC2ReadOnlyAccess, AmazonDynamoDBFullAccess and AWSLambdaVPCAccessExecutionRole.

**Steps to create DynamoDB table**
1.	Go to DynamoDB service and create a table with name ‘ec2_state_change’ and Primary partition key as ‘timestamp’ with number as data type.

![dynamodb](https://user-images.githubusercontent.com/4006576/41896118-5fcf032e-7941-11e8-9c5b-8868a32d5eee.png)

**Steps to create CloudWatch event**
1.	Go to CloudWatch service and create a Rule for ‘EC2 Instance State-change Notification’ and assign the Lambda function created in the above steps.

![cloudwatch](https://user-images.githubusercontent.com/4006576/41896120-63026ed2-7941-11e8-8220-af60752b3f34.png)

# 6. Conclusion
By adhering to the steps discussed in the above article users can seamlessly integrate AWS Cloud Management into TCPWave IPAM to automate DDI in Private DNS or send the state details CSV to external server.

# Assigning Elastic IP to Lambda function
As AWS Lambda provides serverless computing, it doesn’t use static IP Address. So third-party services cannot whitelist the IP as it changes quite frequently. To solve this security concern below are the steps provided to assign a VPC with elastic IP to Lambda function. Once the below set up is done, all the traffic from Lambda service will be NAT’ed to the elastic IP Address. This elastic IP can be white listed by third party vendor.
## Step1
Head over to your AWS VPC dashboard and click on over to list of VPCs. Click on the Create VPC link and enter in a name and CIDR block for the VPC.

![1](https://user-images.githubusercontent.com/4006576/44384723-bfd69f00-a53a-11e8-9cf5-659224697528.png)

## Step2
Go to the Subnets page and create two subnets. One public and one private as shown in below two screenshots.



## Step3
Head over to the Internet Gateway view, click on Create Internet Gateway and tag it with a descriptive tag.




## Step4
Then, click on new internet gateway, and click on Attach to VPC, to attach it to newly created VPC.





## Step5
Now head over to our Route Tables view and click on Create Route Table, giving it a descriptive tag and linking it to the VPC.



## Step6
Then edit this route to point it to new internet gateway. Click on the new route, click on the Routes tab, and click edit. Then add a new route, and set all traffic (0.0.0.0/0) to target our internet gateway and save it.




## Step7
Now, click on Subnet Associations tab, click edit and, by ticking the check box by your public subnet and clicking Save, you will associate this new route to your public subnet.




## Step8
First, take note of your public subnet’s id. Head over the NAT Gateway view and click on Create NAT Gateway. On the creation screen go ahead and paste in your subnet id and click on “Create New EIP.” Elastic IP will be created as below.




## Step9
On the confirmation screen copy nat instance id and go back and edit the default route created when new VPC is created. Click on the default route (you will see the Main column for that route says Yes), click on the Routes tab, and click edit. Then add a new route, and we will set all traffic (0.0.0.0/0) to target nat instance id and save it.


## Step10
Now, click on Subnet Associations tab, click edit and, by ticking the check boxes by public and private subnets and clicking Save, you will associate this new route to the subnets.



## Step11
Create new security group as below from Security groups page in EC2 service.



## Step12
Now, assign the newly created VPC, subnets and security group to the Lambda function as below.











