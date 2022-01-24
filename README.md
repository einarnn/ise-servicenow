# ServiceNow Integration with Cisco ISE

## OVERVIEW

This guide is intended to show how to allow the ServiceNow platform to use information from ServiceNow's CMDB to update endpoint records within ISE. This guide will focus on the following use case:

## USE CASE:

A customer has an inventory of their computers in ServiceNow and would like to ensure that only those computers are allowed access to the network. ServiceNow will therefore inform Cisco ISE of the status of computer objects and ISE will have a policy to either drop ping (default) or allow access (if Computuer is in the Inventory). In this case, the ServiceNow database will use the Status field with a value of "Installed"; to indicate a device as being in the inventory (see screenshot):
![](images/useCaseExample.png)

## REQUIREMENTS:

- ISE v2.4 or later (screenshots are ISE 3.1)
- ServiceNow Instance (Orlando or later)
- ServiceNow MID-Server with access to ISE PSN (TCP 80/443/9060)
  - Instructions on how to install a MID Server can be found [here](https://docs.servicenow.com/bundle/quebec-servicenow-platform/page/product/mid-server/concept/mid-server-installation.html)

## OUTLINE:

1. Create ERS user account within ISE(#_1._CREATE_ERS)
2. Enable ERS Gateway within ISE(#_2._ENABLE_ERS)
3. Create Custom Attributes within ISE(#_3._CREATE_CUSTOM)
4. Create new Policy Conditions and AuthZ Profile(#_4._CREATE_NEW)
5. Verify ERS Functionality via Postman(#_5._Verify_ERS)
6. Create MID Server app / Bind App to MID Server on Prem(#_6._CREATE_NEW)
7. Create the REST API queries in ServiceNow(#_7._CREATE_THE)
  1. Get\_GUI\_By\_MAC(#_Create_a_new)
  2. Get\_Endpoint\_Details(#_Create_new_HTTP)
  3. Put\_Endpoint\_Update(#_Create_new_HTTP_1)
8. Script Automation within ServiceNow(#_8._Script_Automation)
  1. Defining the ServiceNow Script Class(#_Define_the_ServiceNow)
  2. Defining the ServiceNow Business Rule(#_Define_the_ServiceNow_1)
9. Testing the Overall Solution(#_9._Testing_the)

Troubleshooting(#_TROUBLESHOOTING)
Appendix 1(#_APPENDIX_1:_ISE_Helper)
Appendix 2

## Create ERS user account within ISE


## STEP 1. CREATE ERS Account in ISE

From the ISE Admin portal, navigate to Administration -> System -> Admin Access -> Administrators -> Admin Users and click on "Add" -> "Create an Admin User"

Assign this user the "ERS Admin" role and click Submit

![](images/createERSadmin1.png)

![](images/createERSadmin2.png)

## 2. ENABLE ERS Gateway within ISE

Navigate to Administration -> System -> Settings -> API Settings

Switch to the "API Service Settings" tab and enable the ERS (Read/Write) button and click "Save".

NOTE: Though not required for this tutorial, the screenshot below also shows OpenAPI enabled as well.

![](images/enableERSgateway.png)

## 3. Create Custom Attributes in ISE

Navigate to Identity Management -> Settings -> Endpoint Custom Attributes and create three conditions:

In our scenario, we will create three new attributes:
**InventoryStatus** (Boolean) – the "true/false" value for whether or not an endpoint is in the inventory
**SerialNumber** (string) – this value will store the SerialNumber details from records in ServiceNow
**SerialSource** (string) – (optional) this value will store "ServiceNow" as the source to distinguish the record change from other 3rd party integrations or manual modification

![](images/createCustomAttributes1.png)

Click "Save" to save your changes.

To verify that these new attributes have now been added to the ISE endpoint database, navigate to Context Visibility -> Endpoints and select a random mac address. In this case, we've selected our test MAC address of BB:BB:BB:BB:BB:BB

Under Attributes, click the Custom Attributes tab

![](images/CreateCustomAttributes2).png) ![](images/createCustomAttributes3.png)

NOTE: By default these new values will be blank on all endpoints, including the Boolean "InventoryStatus". This will become relevant later in the tutorial.

## 4. Create New Authorization Conditions / AuthZ Profile

Navigate to Policy -> Policy Sets and we will create two rules (a permit and a deny rule) using the value of the new "InventoryStatus" custom attribute that will be populated by ServiceNow. When adding the condition for the each rule, you can search the Attribute field to quickly find the "InventoryStatus" field.

![](images/createAuthZrule1.png)

The resulting rules should look like the following:

![](images/createAuthZrule2.png)

**NOTE: Since the default value for the "InventoryStatus" field for all endpoints is blank, these rules would need to target specific device types (Workstations, etc.) to avoid blocking all devices when this feature is enabled.**

In this example, the AuthZ rule "DROP\_PING" pushes a dACL that drops PING traffic for the workstation. However, this could be customized in a variety of ways including a redirect to a splash page that informs the user their device is not in the inventory.

## 5. Verify ERS Functionality via Postman

Now that ISE rules/policies have been setup, let&#39;s test the functionality using Postman. Reminder that we will need to search for the MAC address of the endpoint for our inventory to obtain the GUID for the record within ISE. This GUID will then be used to gather additional details about the endpoint and/or update the endpoint record.

[Click here for a tutorial on how to use Postman](https://learning.postman.com/docs/getting-started/introduction/)

1. Create a new collection for ISE ERS connectivity.
2. Create a new request inside the collection and name it Endpoint By MAC
  1. Set the value of the request to "GET" with a URL of https://\&lt;ISE\&gt;:9060/ers/config/endpoint


  2. Under Params, add the key &#39;filter&#39; with a value of "mac.EQ.\&lt;your mac\&gt;" which includes a MAC address in your ISE instance.
 ![](RackMultipart20220124-4-nvp436_html_38cca0dbe55f5ea1.png)

The URL now should read: https://\&lt;ISE\&gt;:9060/ers/config/endpoint?filter=mac.EQ.BB:BB:BB:BB:BB:BB

  1. Set the Authorization to Basic Auth and populate the username/pwd details for the ERS Admin created in the ISE instance in [Step 1](#_1._CREATE_ERS)
 ![](RackMultipart20220124-4-nvp436_html_5139542057e1c193.png)
  2. In the Headers section, add the following
 ![](RackMultipart20220124-4-nvp436_html_c5f372ce74211c11.png)
  3. Click the "Send" button. The output should be similar to below.

![](RackMultipart20220124-4-nvp436_html_2f82c5ec99b3dedc.png)

The ID returned is the GUID. We will need to copy this value for the next step.

1. Click the ellipsis next to the existing GET request and create a duplicate, and name it "Endpoint by UID"
  1. Set the URL value to https://\&lt;ISE\&gt;:9060/ers/config/endpoint/\&lt;GUID\&gt;
 ![](RackMultipart20220124-4-nvp436_html_26fc0a20125aaecf.png)
  2. Since you duplicated this request, the Authorization and Headers sections are already populated.
  3. Click the "Send" button. The output should be similar to below:

![](RackMultipart20220124-4-nvp436_html_27810b09e139b401.png)

NOTE: The Custom Attributes fields for this specific endpoint are returned with blank values.

1. Click the ellipsis next to "Endpoint by GUID" and create a duplicate and name it "Update Endpoint by UID"
  1. Set the request type to "PUT" and the URL should remain in the same format as the previous "GET" request (ex. https://\&lt;ISE\&gt;:9060/ers/config/endpoint/\&lt;GUID\&gt;
  2. We will now be sending additional information in the ERS message, so we need to modify the format. Click on the Headers section and add a value of "Content-Type" with value "application/json"
 ![](RackMultipart20220124-4-nvp436_html_70592f9f331d1099.png)

  1. Click on the "Body" section. Here we will add values to update for this endpoint including the "true" value for the InventoryStatus, a test SerialNumber value, and the "ServiceNow" for SerialSource.

![](RackMultipart20220124-4-nvp436_html_44770b948897e209.png)
 {

"ERSEndPoint": {

"id": "ac48cc90-1c7b-11ec-b265-9a9fc6c2c68a",

"customAttributes":{

"customAttributes" :{

"InventoryStatus" : true,

"SerialNumber": 123456,

"SerialSource": "ServiceNow"

}

}

}

}

  1. Click the "Send" button to issue the update. The returned output shows the old values of the record as well as the new values after this update.
 ![](RackMultipart20220124-4-nvp436_html_a5d1284338939594.png)
1. Within the ISE GUI, verify that the updates are reflected in the endpoint (Context Visibility -> Endpoints) ![](RackMultipart20220124-4-nvp436_html_38df2ba47e396699.png)
2. Now that we know the ERS functionality is working, let&#39;s clear the Custom Attribute values (Edit Endpoint-> click "trash" icon next to each value, click Save") for this endpoint to reset it back to normal. ![](RackMultipart20220124-4-nvp436_html_63fb65c379b33274.png)
 We will now perform the ServiceNow steps to do this same ERS update process.

## 6. CREATE NEW MID SERVER APPLICATION / BIND APP TO MID SERVER ON PREM

In ServiceNow, navigate to MID Server -> Applications.

Create the new Application in ServiceNow ("ISE-ERS") and bind it to the MID-Server (ex. "ISE\_mid\_server")

![](RackMultipart20220124-4-nvp436_html_c5f362dc2574fe94.png)

![](RackMultipart20220124-4-nvp436_html_532b5b9189ddcaff.png)

This will be referenced later

![](RackMultipart20220124-4-nvp436_html_7d899ff8b4be731.png)

## 7. CREATE THE REST API QUERIES IN SERVICE NOW

We now need to recreate the API calls we performed in Postman into the ServiceNow API engine. To do this, navigate to Outbound -> REST Message and click "New". This will be the basis of all ISE queries so name it something accordingly:

Name: ISE-SVR

Accessible from: This application only
 Authentication – Type: Basic
 Endpoint: [https://\&lt;ISE\&gt;:9060/ers/config](https://ise31a.securitydemo.net:9060/ers/config)

![](RackMultipart20220124-4-nvp436_html_6284f44cf5d2867c.png)

Click the search button next to Basic Auth Profile. Click "New" to create a new credential.

![](RackMultipart20220124-4-nvp436_html_e78e8aa30eec7fe4.png)

Here you will put in the ERS credentials that were created in ISE during step 1. Then click submit

![](RackMultipart20220124-4-nvp436_html_cb0ab1fe9f5ab4f2.png)

![](RackMultipart20220124-4-nvp436_html_7d899ff8b4be731.png)

Click on the HTTP Request tab and add the key "Accept" with value "application/json"
 ![](RackMultipart20220124-4-nvp436_html_e26bc1c51b7d4e97.png)

Now that the basic message format is created, we will create the individual calls:

1.
### Create a new HTTP Method called "Get\_GUI\_By\_MAC"

  1. Set the Endpoint HTTP method to be https://\&lt;ise\&gt;:9060/ers/config/endpoint
  2. Set Authentication to "Inherit from parent" ![](RackMultipart20220124-4-nvp436_html_3e76c71c35c3380b.png)
  3. Click the HTTP Request tab. For Use MID Server, click the search button to select your MID Server instance within ServiceNow that will run this request.
 ![](RackMultipart20220124-4-nvp436_html_5336c8888ed7d677.png)
  4. Since HTTP headers are inherited from parent, we don&#39;t need to define them in this request.
  5. Under the HTTP Query field, double-click to Insert a new row. For the name type "filter" and for the value, type "mac.EQ.${mac}". ![](RackMultipart20220124-4-nvp436_html_b6ea98b024a27b51.png)

This value will reference a variable that we will now define.

  1. Under Variable Substitutions, click New. In the next screen, enter "mac" for the name, leave Escape Type to "no escaping" and set the test value to our example MAC address of BB:BB:BB:BB:BB, and then click Submit. ![](RackMultipart20220124-4-nvp436_html_7d94a543822f4b50.png)
 ![](RackMultipart20220124-4-nvp436_html_a3b727da38c79c3c.png)
  2. Now we can perform a test run of the MID server communication to the ISE PAN via ERS call. Under Related Links, click the "Test" button.

![](RackMultipart20220124-4-nvp436_html_ac0f9376b9c1e75d.png)

After a few moments, a new screen displays the Test Results including the GUID value (copy this). ![](RackMultipart20220124-4-nvp436_html_dc09e6e449ee4fa8.png)

Find the GUID for the MAC is returned, "9b3cafc….". Copy this value for the next step.

1.
### Create new HTTP Method Called "Get\_Endpoint\_Details"

  1. Set the following values (replicating steps from the previous method:

Name: GET\_Endpoint\_Details
 HTTP Method: GET
 Endpoint: https://\&lt;ise\&gt;:9060/ers/config/endpoint/${GUIDendpoint}
 Authentication Type: Inherit from Parent
 ![](RackMultipart20220124-4-nvp436_html_1556044edf63d0f3.png)

  1. Click the HTTP Request tab and ensure the appropriate MID server is selected.

![](RackMultipart20220124-4-nvp436_html_5336c8888ed7d677.png)
 No other settings are needed here

  1. Under variable Substitutions, click New. In the next screen, enter the following:

Name: GUIDendpoint

Test value: \&lt;sample GUID for example MAC from above\&gt;
 ![](RackMultipart20220124-4-nvp436_html_835c3c63a1100f25.png)

  1. Click the "Test" button under Related Links and confirm the output looks like the following:

![](RackMultipart20220124-4-nvp436_html_8ba1b12dbb8e8d40.png)

1.
### Create new HTTP Method called "PUT\_Endpoint\_Update"

  1. Set the following values (replicating steps from the previous method:

Name: "PUT\_Endpoint\_Update"
 HTTP Method: "PUT"

Endpoint: https://\&lt;ise\&gt;:9060/ers/config/endpoint/${GUIDendpoint}
 Authentication Type: Inherit from Parent

  1. Click the HTTP Request tab and ensure the appropriate MID server is selected.

![](RackMultipart20220124-4-nvp436_html_5336c8888ed7d677.png)

Under HTTP Headers, add a new field of "Content-Type" and value "application/json"

![](RackMultipart20220124-4-nvp436_html_53b5496342adbee4.png)

  1. Under HTTP Query Parameters, add the following into the "Content" window and click Update:

{

"ERSEndPoint": {

"id": "${GUIDendpoint}",

"customAttributes":{

"customAttributes" :{

"InventoryStatus" : ${InventoryStatus},

"SerialNumber": ${SerialNumber},

"SerialSource": "${SerialSource}"

}

}

}

}

![](RackMultipart20220124-4-nvp436_html_e99fdfc54eed5cb3.png)

  1. Scroll down to the Variable Substitutions section. Here we will define (4) variables that will be used in this query (referenced in the code from the previous step):
    1. Name: GUIDendpoint, value = GUID value from previous query ![](RackMultipart20220124-4-nvp436_html_effa03cdd7b096b.png)
    2. Name: Inventory Status, value = true
    3. Name: SerialNumber, value = 987654 (Serial # different than Postman example)
    4. Name: SerialSource, value = ServiceNow

The end result should look like this ![](RackMultipart20220124-4-nvp436_html_3c3690ef9112ec8.png)

  1. Click the "Test" button under Related Links. The output should look like the following:
 ![](RackMultipart20220124-4-nvp436_html_da12e7f6b7dfcd9f.png)
1. To verify the integration worked correctly, once again navigate to Context Visibility -> Endpoints and pull up the same endpoint to see the attributes updated:
 ![](RackMultipart20220124-4-nvp436_html_4b1e7714c8484302.png)

## 8. Script Automation within ServiceNow

Now that we have successfully tested the REST API calls from ServiceNow to update ISE, we need to automate this process based on ServiceNow workflows. To do this, we need to build two components:

1) Script Class – accepts details from the CMDB, calls the REST messages, and parses the output

2) Business Rule – define what action within ServiceNow will trigger Script Class

1.
## Define the ServiceNow Script Class

Navigate to "System Definition-> Script Includes" and click "New", name it "ISE\_Helper" , select the Accessible from "This Application scope only", check the box for "Active", and then use the script from the [appendix](#_APPENDIX_1:_ISE_Helper). The result should look similar to the screenshot below:

 ![](RackMultipart20220124-4-nvp436_html_5292315580e2faa1.png)

1.
## Define the ServiceNow Business Rule

In this scenario, we want the ServiceNow process to run whenever a new Network Adapter entry is added to the ServiceNow CMDB. Other options can be selected per use case.

Navigate to "Business Rules" and click "New". Populate the following fields:
 Name: ISE\_Network\_Adapter

Active: Enabled

Advanced: Enabled

When to Run: When "After" an "Insert"

![](RackMultipart20220124-4-nvp436_html_13740b9ece3c134c.png)

Click on the "Advanced" tab and add the following script (Appendix 2) that will use the information from the Network\_Adapter table, send it to the Script Class, and then return the results.

## 9. Testing the Overall Solution

Using our test MAC address from before, let&#39;s go back to the ISE dashboard and remove the custom attributes that exist for our test device, BB:BB:BB:BB:BB:BB.

Navigate to Context Visibility -> Endpoints -> select MAC address -> edit endpoint. Open Custom Attributes, and click the small "trash can" icon for each of the three attributes to clear them. Then click Save.

![](RackMultipart20220124-4-nvp436_html_63fb65c379b33274.png) ![](RackMultipart20220124-4-nvp436_html_40061877e0ebc677.png)

In ServiceNow, navigate to the Configuration -> Computers window to view the active endpoints in the CMDB.

![](RackMultipart20220124-4-nvp436_html_855dff75b29b8a46.png)

For our example, we will use "Computer1". Click on the computer name to open the Computer record.

![](RackMultipart20220124-4-nvp436_html_cfa29a16adcef875.png)

NOTE: This computer is currently in an "Installed" status and has a serial number of "123456".

Scroll down to the Network Adapters section. This section shows all of the associated MAC addresses with this workstation. In this case, there are no MAC addresses associated to this endpoint, so we will add one.

![](RackMultipart20220124-4-nvp436_html_354275bd5ff634dd.png)

Click "New" and input the value of "BB:BB:BB:BB:BB:BB" for the MAC address field. The other fields are not necessary to modify at this time. Click the "Submit" button.

![](RackMultipart20220124-4-nvp436_html_82d8bd00ccfac99d.png)

After clicking "Submit", recall that this is an Insertion into the Network Adapter table, so therefore the Business Rule will be triggered on the backend. After a few moments, you will see the result of the API calls:

![](RackMultipart20220124-4-nvp436_html_aa7b50b32568192a.png)

In this case, we see the "PASS" for the update of the MAC address of "BB:BB:BB:BB:BB:BB".

Again, verify in ISE by going to Context Visibility -> Endpoints and view the details of that MAC address.

![](RackMultipart20220124-4-nvp436_html_38df2ba47e396699.png)

As soon as the device re-authenticates, it will now hit the Policy Condition that we created earlier that will check the "InventoryStatus = true". NOTE: This does not happen automatically. The client re-auth can either be initiated by the client (disconnect/reconnect, or timed re-auth), or manually triggered CoA by an admin through the ISE Admin console.

In this case, we were using a test MAC address so we can&#39;t show the live logs of this MAC, but here is an example of an actual device who&#39;s access was modified by this same process after re-authentication:

![](RackMultipart20220124-4-nvp436_html_b7c7a9e6bd9224b1.png)

## TROUBLESHOOTING

Scenario 1: ServiceNow updates ISE fields, but sets InventoryStatus to "False" instead of "True".

**SOLUTION:** Verify that the computer object within ServiceNow has a Status value set to "Installed". The Script Class is configured to utilize this property when updating NICs. This means that if you have the variable set to another value, (ex. Absent), then ServiceNow will interpret this as an object that is not in the inventory. For example, the Computer object below would update SerialNumber details, but set Inventory Status to "False"

![](RackMultipart20220124-4-nvp436_html_7c624d84c14c4270.png) ![](RackMultipart20220124-4-nvp436_html_2d3737e074540c22.png)

Also, the message dialog after an ISE update should reflect this behavior with "Inventory Status: FAIL"

![](RackMultipart20220124-4-nvp436_html_a847c00a036c116.png)

Scenario 2: Receive HTTP response other than 200.

**SOLUTION** : Verify configuration of required queries/headers for REST message

Scenario 3: Receive "No response" for ECC Message Request within ServiceNow

![](RackMultipart20220124-4-nvp436_html_1f1024e8f1dd7554.png)

**SOLUTION:** Verify that the MID Server associated to the ISE REST messages is currently in a running state.

Navigate to Outbound -> REST Message -> ISE\_Helper -> Get\_GUI\_By\_MAC method

Click on the "HTTP Request Tab" and next to USE MID Server, hover over the "Info" button to see the current status of the MID Server.

![](RackMultipart20220124-4-nvp436_html_5fda557a9a6a07eb.png)

If the Server Status is "Down" but you have verified the box is online, make sure the MID Service is running on the Server (Start -> Services -> ServiceNow MID Server\_ISE\_\&lt;name\&gt;)

![](RackMultipart20220124-4-nvp436_html_6c4087dd1dd1084c.png)

**If it is not running, start the process. Wait a few moments, refresh the ServiceNow page and check the status again:**

![](RackMultipart20220124-4-nvp436_html_9f73f1526b4033db.png)