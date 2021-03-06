# Enabling AutoDiscover to Configure Endpoints<a name="autodiscover"></a>

AutoDiscover enables you to easily configure Microsoft Outlook and mobile clients with only your email address and password\. The service also maintains a connection to Amazon WorkMail and updates local settings whenever endpoint or settings changes are made\. In addition, AutoDiscover enables your client to use additional Amazon WorkMail features, such as the Offline Address Book, Out\-of\-Office Assistant, and the ability to view free/busy time in Calendar\. 

The client performs the following AutoDiscover phases to detect the server endpoint URLs:
+ Phase 1: The client performs an SCP lookup against the local Active Directory\. If your client isn’t domain\-joined, AutoDiscover skips this step\.
+ Phase 2: The client sends a request to the following URLs and validates the results\. These endpoints are only available using HTTPS\.
  + https://company\.tld/autodiscover/autodiscover\.xml 
  + https://autodiscover\.company\.tld/autodiscover/autodiscover\.xml
+ Phase 3: The client performs a DNS lookup to autodiscover\.company\.tld and sends an unauthenticated GET request to the derived endpoint from the user’s email address\. If the server returns a 302 redirect, the client resends the AutoDiscover request against the returned HTTPS endpoint\. 

If all of these phases fail, the client can’t be configured automatically, and you must set up the client manually\. For information about manually configuring mobile devices, see [Manually Connect Your Device](https://docs.aws.amazon.com/workmail/latest/userguide/manually_connect_device.html)\.

When you set up your domain in Amazon WorkMail, you are prompted to add the AutoDiscover DNS record\. This enables the client to perform phase 3 of the AutoDiscover process\. However, these steps don't work for all mobile devices, such as the stock Android email app, and you may need to set up AutoDiscover phase 2 manually\.

There are two ways you can set up AutoDiscover phase 2 for your domain:
+ By using Route 53 and Amazon CloudFront \(recommended\)
+ By setting up an Apache web server with a reverse proxy

**To enable AutoDiscover phase 2 with Route 53 and CloudFront**
**Note**  
The following steps show how to proxy https://autodiscover\.company\.tld/autodiscover/autodiscover\.xml\. To proxy https://company\.tld/autodiscover/autodiscover\.xml, remove the "autodiscover\." prefix from the domains in the following steps\.  
For more information about applicable pricing, see [Amazon CloudFront Pricing](https://aws.amazon.com/cloudfront/pricing/) and [Amazon Route 53 Pricing](https://aws.amazon.com/route53/pricing/)\.

1. Get an SSL certificate for autodiscover\.company\.tld and upload it to IAM\. For more information, see [Working with Server Certificates](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_server-certs.html)\.

1. Create a new CloudFront distribution\.

   1. Open the CloudFront console at [ https://console\.aws\.amazon\.com/cloudfront/](https://console.aws.amazon.com/cloudfront/)\.

   1. Choose **Create Distribution**, **Web** and **Get Started**\.

   1. Fill in the following values for **Origin Settings**:
      + **Origin Domain Name**: autodiscover\-service\.mail\.us\-east\-1\.awsapps\.com, autodiscover\-service\.mail\.eu\-west\-1\.awsapps\.com, or autodiscover\-service\.mail\.us\-west\-2\.awsapps\.com 
      + **Origin Protocol Policy**: Match Viewer
**Note**  
Leave **Origin path** blank, and do not change the auto\-populated value for **Origin ID**\.

   1. Fill in the following values for **Default Cache Behavior Settings**:
      + **Viewer Protocol Policy**: HTTPS Only
      + **Allowed HTTP Methods**: GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE 
      + **Cache Based on Selected Request Headers**: All 
      + **Forward Cookies**: All 
      + **Query String Forwarding and Caching**: None \(Improves Caching\) 
      + **Smooth Streaming**: No 
      + **Restrict Viewer Access**: No 

   1. Fill in the following values for **Distribution Settings**:
      + **Price Class**: Use only US, Canada, and Europe
      + **Alternate Domain Names \(CNAMEs\)**: autodiscover\.company\.tld \(or company\.tld\)
      + **SSL Certificate**: Custom SSL Certificate \(stored in IAM\)
      + **Custom SSL Client Support**: All Clients
**Note**  
Leave **Default Root Object** blank\.
      + **Logging**: Choose **On** or **Off**
      + **Comment**: AutoDiscover type2 for autodiscover\.company\.tld 
      + For **Distribution State**, choose **Enabled**

1. In Route 53, connect the CloudFront distribution to DNS:
**Note**  
These steps assume that the DNS record for company\.tld is hosted in Route 53\.

   1. In the Route 53 console, choose **Hosted Zones** and **company\.tld**\. 

   1. Choose **Create Record Set**, and then fill in the following fields:
      + **Name**: autodiscover\.company\.tld
      + **Type**: A \- IPv4 address
      + **Alias**: Yes
      + **Alias Target**: The CloudFront distribution created above
**Note**  
If the CloudFront distribution created above is not present, wait a while and try again later\. Change propagation for new CloudFront endpoints in Route 53 might take up to 1 hour\. 
      + **Evaluate Target Health**: No

   1. Choose **Create**\.

**To enable AutoDiscover phase 2 with an Apache web server**

1. Configure the following two directives on an SSL\-enabled Apache server: 

   ```
   SSLProxyEngine on ProxyPass /autodiscover/autodiscover.xml
   https://autodiscover-service.mail.REGION.awsapps.com/autodiscover/autodiscover.xml
   ```

1. If they are not already enabled, enable the following Apache modules:
   + proxy
   + proxy\_http
   + socache\_shmcb
   + ssl

1. Confirm that the endpoint is SSL\-enabled and configured correctly\.

## AutoDiscover Phase 2 Troubleshooting<a name="troubleshooting"></a>

To make a basic unauthorized request, create an unauthenticated POST request to the AutoDiscover endpoint and see if it returns a “401 unauthorized” message:

```
$ curl -X POST -v https://autodiscover.''company.tld''/autodiscover/autodiscover.xml
...
HTTP/1.1 401 Unauthorized
```

If the basic request is unsuccessful and returns a “401 unauthorized” message, run a real request that a mobile device would issue\.

To do this, first create a request\.xml file with the following XML content: 

```
<?xml version="1.0" encoding="utf-8"?>
<Autodiscover xmlns="http://schemas.microsoft.com/exchange/autodiscover/mobilesync/requestschema/2006">
     <Request>
            <EMailAddress>testuser@company.tld</EMailAddress>
            <AcceptableResponseSchema>
             http://schemas.microsoft.com/exchange/autodiscover/mobilesync/responseschema/2006
            </AcceptableResponseSchema>
     </Request>
</Autodiscover>
```

Second, make the request\.

```
$ curl -d @request.xml -u testuser@company.tld -v https://autodiscover.company.tld/autodiscover/autodiscover.xml
Enter host password for user 'testuser@company.tld':
<?xml version="1.0" encoding="UTF-8"?>
<Autodiscover xmlns="http://schemas.microsoft.com/exchange/autodiscover/responseschema/2006" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
<Response xmlns="http://schemas.microsoft.com/exchange/autodiscover/mobilesync/responseschema/2006">
    <Culture>en:us</Culture>
    <User>
        <DisplayName>User1</DisplayName>
        <EMailAddress>testuser@company.tld</EMailAddress>
    </User>
    <Action>
        <Settings>
            <Server>
                <Type>MobileSync</Type>
                <Url>https://mobile.mail.us-east-1.awsapps.com/Microsoft-Server-ActiveSync</Url>
                <Name>https://mobile.mail.us-east-1.awsapps.com/Microsoft-Server-ActiveSync</Name>
            </Server>
        </Settings>
    </Action>
</Response>
```

If the response output is similar, your AutoDiscover endpoint is configured correctly\.