# Canary Deployment - CAA DCP Integration

## What is canary deployment?

**Canary release** is a technique to reduce the risk of introducing a new software version in production by slowly rolling out the change to a small subset of users before rolling it out to the entire infrastructure and making it available to everybody.

Similar to  a **Blue Green Deployment**, we start by deploying the new version of your software to a subset of your infrastructure, to which no users are routed.

![Diagram 1](https://martinfowler.com/bliki/images/canaryRelease/canary-release-1.png "New enviornment created")

As we gain more confidence in the new version, you can start releasing it to more servers inyour infrastructure and routing more users to it. A good practice to rollout the new version is to repurpose our existing infrastructure.

In a canary release the miration pase lasts untill all the isers have been routed to the new version. At that point, we can decomisson the old infrastructure. If we find problems with the new version, the rollback strategy is simply to reroute users back to the old version untill we have fixed the problem.

![Diagram 1](https://martinfowler.com/bliki/images/canaryRelease/canary-release-3.png "New enviornment created")

By slowly ramping up the load, you can monitor and capture metrics about how the new version impacts the production environment. This is an alternative approach to creating an entirely separate capacity testing environment, because the environment will be as production-like as it can be, this is a benefit of using canary deployment techniques.

## Canary Deployment in CAA-DCP perspective.

Here in this senario we will be using canary deployment using AWS' **Route 53**.

### Route 53

Route 53 is a highly available and scalable cloud DNS web service

Amazon Route 53 effectively connects user requests to infrastructure running in AWS – such as Amazon EC2 instances, Elastic Load Balancing load balancers, or Amazon S3 buckets – and can also be used to route users to infrastructure outside of AWS.

### **Steps to follow (Previously):**

1. Write the jenkins file script to 'Deploy the application on the staging enviornment using ansible playbook'

    ```groovy
    stage ('Deploying application on Staging Environment with ansible') {
      when {
            expression { params.DeploymentType == 'BlueGreen' || params.DeploymentType == 'Canary' }
      }
      steps {
        sh "ssh ec2-user@10.0.55.78 ansible-playbook -i /home/ec2-user/ansible/STG/playbooks/hosts /home/ec2-user/ansible/STG/playbooks/apache-tomcat.yml"  
      }
    }
    ```

    The above ansible playbook _apache-tomcat.yml_ contains the ansible script to provison a tomcat server where the war file will be deployed.

2. Write the jenkinsfile script for 'Updating the Record set in Route53 for Staging Environment'.

   ```groovy
    stage ('Updating the Record set in Route53 for Staging Environment') {
        when {
            expression { params.DeploymentType == 'BlueGreen' || params.DeploymentType == 'Canary' }
        }
        steps {
            sh "ssh ec2-user@10.0.55.78 python /home/ec2-user/recordset.py Name STGLoadBalancer staging-app"

            sleep(time:30,unit:"SECONDS")
        }
    }
    ```

    The above mentioned _**recordset.py**_ file contains the pythons script to write the recordset to a hostedzone in **Route 53**. This script use AWS python SDK BOTO3.  the script is as shown below.

    The Python script can be divided into 3 main parts:
    * **Getting loadbalancer DNS Name:** this method is used to get the name of the required loadbalancer.

        ```python
        applb = elb_client.describe_load_balancers()
        ARNlist = []
        for var in range(len(applb['LoadBalancers'])):
                ARNlist.append(applb['LoadBalancers'][var]['LoadBalancerArn'])
        finalarn = "initalvalue"
        elbdns = "initalvalue"
        for eachARN in ARNlist:
                res=elb_client.describe_tags(ResourceArns=[eachARN])
                for count in range(len(res['TagDescriptions'][0]['Tags'])):
                        keyname= (res['TagDescriptions'][0]['Tags'][count]['Key'])
                        keyvalue=(res['TagDescriptions'][0]['Tags'][count]['Value'])
                        if keyname == checkKey and keyvalue == checkValue:
                                finalarn=eachARN
                                break;
        elbResponse = elb_client.describe_load_balancers( LoadBalancerArns=[finalarn])
        elbdns=elbResponse['LoadBalancers'][0]['DNSName']
        return elbdns
        ```

    * **Creating a recordset of type 'A':** This method is used to create a new recordset if it doesn't exists already or update an existing one.

        ```python
        def createRecordSetA(elbdns,recordname):
        recordsetname = str(recordname)+".devops.oneweb1.com"
        recordResponse = route53_session.change_resource_record_sets(
                HostedZoneId = 'Z1Q9HXLZLE5ZNW',
                ChangeBatch = {
                'Comment': 'Upsert a record',
                'Changes': [
                    {
                        "Action": "UPSERT",
                                "ResourceRecordSet": {
                                        "Name": recordsetname,
                                        "Type": "A",
                                        "AliasTarget": {
                                                "HostedZoneId": "Z35SXDOTRQ7X7K",
                                                "DNSName": elbdns,
                                                "EvaluateTargetHealth": True
                                    }

                            }
                    }

            ]})
        print ("A record Created\n")
        print (recordResponse)
        ```

    * **Creating a recordset of type 'CNAME':** This methood is used to create a new CNAME record set if it already doesn't exist or update an already existing one.

        ```python
        def createRecordSetCname(recordname):
        recordsetname="www."+str(recordname)+".devops.oneweb1.com"
        recordsetvalue=str(recordname)+".devops.oneweb1.com"
        recordResponse=route53_session.change_resource_record_sets(
                HostedZoneId='Z1Q9HXLZLE5ZNW',
                ChangeBatch={
                'Comment': 'Upsert a record',
                'Changes': [
                    {
                        "Action": "UPSERT",
                                "ResourceRecordSet": {
                                        "Name": recordsetname,
                                        "Type": "CNAME",
                        "TTL" : 30,
                                        "ResourceRecords": [{
                                            "Value": recordsetvalue
                                    }]

                            }
                    }

            ]})
        print ("CNAME record Created\n")
        print (recordResponse)
       ```

3. Write the jenkinsfile script for 'Creating the inital Record for canary deployment'.

   ```groovy
   stage ('Proceed with Canary Deployment') {
        when {
            expression { params.DeploymentType == 'Canary' }
        }
        steps {
            input 'Proceed With Canary Deployment?'
            sh "ssh ec2-user@10.0.55.78 python /home/ec2-user/Initial_Canary_Record_creation.py production-app staging-app 255 0"
            sleep(time:30,unit:"SECONDS")
        }
    }
   ```

   The above mentioned _**Initial_Canary_Record_Creation.py**_ contains the python script to create weigted recordsets to route traffic to the application accordingly.

   The above script passes 4 arguments to the file. The last two arguments are the weight value that is assigned to the routes so that Route53 can according route traffic.

   The python script consists of the the following methods:

   * **Creation of recordset for old production enviornment:** This method is used to create a record   set for the old production enviornment.

        ```python
        def productionOld_record_Creation(recordname_old_prod,oldprodWeight):
        recordname_old_prod=str(recordname_old_prod)+".devops.tameurcloud.com."
        oldprodWeight=int(oldprodWeight)
        recordDetails=dict()
        recordDetails=Get_Details_of_Records(recordname_old_prod)
        delete_DNS_record_A(recordDetails,recordname_old_prod)
        res=route53_session.change_resource_record_sets(
                HostedZoneId='Z1MBGEEVRECU0O',
                ChangeBatch={
                'Comment': 'Upsert a record',
                'Changes': [
                {
                        "Action": "UPSERT",
                                "ResourceRecordSet": {
                                        "Name": recordname_old_prod,
                                        "Type": "A",
                                        "SetIdentifier": "OldProd",
                                        "Weight": oldprodWeight,
                                        "AliasTarget": {
                                                "HostedZoneId" : recordDetails['HostedZoneId'],
                                                "DNSName": recordDetails['DNSName'],
                                                "EvaluateTargetHealth": recordDetails['EvaluateTargetHealth']
                                }

                        }
                }

        ]})
        ```

   * **Creation of recordset for new production enviornment:** This method is used to create a record set for the new production enviornment.

        ```python
        def productionNew_record_Creation(recordname_new_prod,recordname_old_prod,newprodWeight):

        recordname_new_prod=str(recordname_new_prod)+".devops.tameurcloud.com."
        recordname_old_prod=str(recordname_old_prod)+".devops.tameurcloud.com."
        newprodWeight=int(newprodWeight)
        recordDetails=dict()
        recordDetails=Get_Details_of_Records(recordname_new_prod)

        res=route53_session.change_resource_record_sets(
                HostedZoneId='Z1MBGEEVRECU0O',
                ChangeBatch={
                'Comment': 'Upsert a record',
                'Changes': [
                {
                        "Action": "UPSERT",
                                "ResourceRecordSet": {
                                        "Name": recordname_old_prod,
                                        "Type": "A",
                                        "SetIdentifier": "NewProd",
                                        "Weight": newprodWeight,
                                        "AliasTarget": {
                                                "HostedZoneId": recordDetails['HostedZoneId'],
                                                "DNSName": recordDetails['DNSName'],
                                                "EvaluateTargetHealth": recordDetails['EvaluateTargetHealth']
                                }

                        }
                }

        ]})
        ```

   * **Getting details of all the DNS Record from Route53:** This method is used to get all the DNS records from the record set using the AWS SDK.

        ```python
        def Get_Details_of_Records(recordname):

            all_record_response=route53_session.list_resource_record_sets(HostedZoneId='Z1MBGEEVRECU0O')
            recordDetails=dict()
            for var in range(len( all_record_response['ResourceRecordSets'])):
                    if all_record_response['ResourceRecordSets'][var]['Name'] == recordname :
                            if all_record_response['ResourceRecordSets'][var]['Type'] == 'A':
                                    recordDetails['HostedZoneId']=all_record_response['ResourceRecordSets'][var]['AliasTarget']['HostedZoneId']
                                    recordDetails['DNSName']=all_record_response['ResourceRecordSets'][var]['AliasTarget']['DNSName']
                                    recordDetails['EvaluateTargetHealth']=all_record_response['ResourceRecordSets'][var]['AliasTarget']['EvaluateTargetHealth']
                                    try:
                                            recordDetails['Weight']=all_record_response['ResourceRecordSets'][var]['Weight']
                                    except:
                                            recordDetails['Weight']="NO_Weight"
                                    print (recordDetails)
                                    break
            return recordDetails
        ```

   * **Deleting the DNS record of the old production enviornment:** This method will delete the DNS record of the old production enviornment which we dont need.

        ```python
        def delete_DNS_record_A(recordDetails,recordname_old_prod):
        res=route53_session.change_resource_record_sets(
                HostedZoneId='Z1MBGEEVRECU0O',
                ChangeBatch={
                'Comment': 'Delete the record',
                'Changes': [
                 {
                  "Action": "DELETE",
                        "ResourceRecordSet": {
                                "Name": recordname_old_prod,
                                "Type": "A",
                                "AliasTarget": {
                                        "HostedZoneId": recordDetails['HostedZoneId'],
                                        "DNSName": recordDetails['DNSName'],
                                         "EvaluateTargetHealth": recordDetails['EvaluateTargetHealth']

                                }

                        }
                }

        ]})
        ```

4. Write a Jenkinsfile to 'Divide the traffic 50/50 to two different enviornments'.

    ```groovy
    stage ('Routing 50/50 Traffic as part of Canary Deployment') {
        when {
            expression { params.DeploymentType == 'Canary' }
        }
        steps {
            input 'Proceed With 50/50 Routing Traffic?'
            sh "ssh ec2-user@10.0.55.78 python /home/ec2-user/Canary-Change-weight.py production-app OldProd NewProd 128 128"
            sleep(time:30,unit:"SECONDS")
        }
    }
    ```

    The above script runs a _**canary-change-weight.py**_ file, which consists of the script for canging the weight value of the weight attribute of the route53 and hence routing the traffic to the required enviornment.

    The python script can be divided into these methods:
    * **Get the details of the records:** This method is used to get the details of the record of all the available resources. and identify the old production en

        ```python
        def get_Details_of_Records(recordname,oldprodWeight,newprodWeight,identifer_old_prod,identifer_new_prod):
        recordname=str(recordname)+".devops.tameurcloud.com."
        all_record_response=route53_session.list_resource_record_sets(
                HostedZoneId='Z1MBGEEVRECU0O')

        for var in range(len( all_record_response['ResourceRecordSets'])):
                recordDetails=dict();
                try:
                        if all_record_response['ResourceRecordSets'][var]["SetIdentifier"] :
                                if all_record_response['ResourceRecordSets'][var]["SetIdentifier"] == identifer_old_prod :
                                        recordDetails['HostedZoneId']=all_record_response['ResourceRecordSets'][var]['AliasTarget']['HostedZoneId']
                                        recordDetails['DNSName']=all_record_response['ResourceRecordSets'][var]['AliasTarget']['DNSName']
                                        recordDetails['EvaluateTargetHealth']=all_record_response['ResourceRecordSets'][var]['AliasTarget']['EvaluateTargetHealth']
                                        recordDetails['SetIdentifier']=all_record_response['ResourceRecordSets'][var]['SetIdentifier']

                                        print (recordDetails)
                                        production_record_Creation(recordname,oldprodWeight,recordDetails)

                                if all_record_response['ResourceRecordSets'][var]["SetIdentifier"]== identifer_new_prod :
                                        recordDetails['HostedZoneId']=all_record_response['ResourceRecordSets'][var]['AliasTarget']['HostedZoneId']
                                        recordDetails['DNSName']=all_record_response['ResourceRecordSets'][var]['AliasTarget']['DNSName']
                                        recordDetails['EvaluateTargetHealth']=all_record_response['ResourceRecordSets'][var]['AliasTarget']['EvaluateTargetHealth']
                                        recordDetails['SetIdentifier']=all_record_response['ResourceRecordSets'][var]['SetIdentifier']

                                        print (recordDetails)
                                        production_record_Creation(recordname,newprodWeight,recordDetails)
                except :
                        print("Not the concerned record")

        ```

    * **Allocating weight to the record set of records for different deployment enviornments:** This method is going to assign 50% of the traffic to the old server and another 50% of the traffic to the new server.

        ```python
        def production_record_Creation(recordname,recordWeight,recordDetails):
            recordWeight=int(recordWeight)
            res=route53_session.change_resource_record_sets(
                    HostedZoneId='Z1MBGEEVRECU0O',
                    ChangeBatch={
                    'Comment': 'Upsert a record',
                    'Changes': [
                    {
                            "Action": "UPSERT",
                                    "ResourceRecordSet": {
                                            "Name": recordname,
                                            "Type": "A",
                                            "SetIdentifier": recordDetails['SetIdentifier'],
                                            "Weight": recordWeight,
                                            "AliasTarget": {
                                                    "HostedZoneId" : recordDetails['HostedZoneId'],
                                                    "DNSName": recordDetails['DNSName'],
                                                    "EvaluateTargetHealth": recordDetails['EvaluateTargetHealth']
                                    }

                            }
                    }

            ]})
        ```

5. Write the jenkinsfile for routing all of the traffic to the new enviornment.
   
   ```groovy
    stage ('Routing 100% Traffic to New Environment') {
        when {
            expression { params.DeploymentType == 'Canary' }
                }
        steps {
            input 'Proceed With 100% Routing Traffic to new environment?'
            sh "ssh ec2-user@10.0.55.78 python /home/ec2-user/Canary-Change-weight.py production-app OldProd NewProd 0 255"  
            sleep(time:30,unit:"SECONDS")
        }
    }
   ```

    Here we are using te same script _**canary-change-weight**_ but passing reducing the weightage to the old enviornmet to 0 and routing 100% of the traffic to the new Enviornment.

This is how we are using canary deployment methods using BOTO3 SDK.

**Source:**

BOTO3 SDK Documentation: https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/route53.html