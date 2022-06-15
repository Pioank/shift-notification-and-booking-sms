# shift-notification-and-booking-sms

## Description

Companies might have extra shifts for their employees, which are considered overtime and are not allocated. These shifts are usually communicated in advance and employees who are seeking to generate extra income can book them. There are two parts in this process:

- The company communicating the available shifts to employees who are interested
- Employees confirming, which of the available shifts want

Such use cases involve employees who hold manual posts with little access to the internet. SMS is the preffered channel to communicate available shifts as well as receive the employees' requests. Using SMS ensures that everyone can receive and book available shifts with no accessibility issues. For companies that have a call center, this solution can function as a first line application and still give the option to its users to receive a call from an agent.

## Solution

The solution in this repository covers both parts of the process described above.

**Communicating available shifts to employees who are interested:**
New shifts should be recorded in a data base, along with their status (available / taken) and other information that can be useful when sending the message. This solution uses a DynamoDB table called **shift_status** to store the shifts with the **key = shift_id**. When new shifts are added, marketers or an automated process, will create a new line in a second DynamoDB table called **shift_campaigns**  with **key = campaign_id**. The **shift_campaigns** DynamoDB table comprises from the **key** and a free text field containing a list of the available shifts. Users can add any information about the shift they want to as long as they contain the **shift_id**.

Whenever a new line is created in the **shift_campaigns** DynamoDB table, DynamoDB-Streams invokes an AWS Lambda function that creates an Amazon Pinpoint SMS Campaign using a dynamic segment, which includes all employees interested in extra shifts. Note that it is assumed you have already imported your employee data as Pinpoint SMS endpoints and have created a dynamic semgent that includes all employees interested in extra shifts.

The message sent to the employees contains the **shifts**, which are being retrieved from the **shifts_campaigns** DynamoDB table and a static part to inform recipients how to reply. See example below:
"Available shifts: ' + **shifts** + '. To secure a shift reply by typing the codes separated by comma e.g. XYZ123, XYZ222. Text REQUEST to receive all available shifts or STOP to stop receiving such notifications. If you want to speak to an agent text AGENT and what you need help with."
For companies that don't have or don't want to integrate with their contact center, the message looks like this:
"Available shifts: ' + shifts + '. To secure a shift reply by typing the codes separated by comma e.g. XYZ123, XYZ222. Text REQUEST to receive all available shifts or text STOP to stop receiving such notifications"

### SMS campaign logic
![sms-campaign-logic](https://github.com/Pioank/shift-notification-and-booking-sms/blob/main/assets/sms-campaign-logic.PNG)

### SMS campaign architecture
![sms-campaign-architecture](https://github.com/Pioank/shift-notification-and-booking-sms/blob/main/assets/sms-campaign-architecture.PNG)

**Employees confirming, which of the available shifts want**
Employees can confirm, which shifts want to sign-up for by sending an SMS with the shift ids separated by comma. If they cannot remember the shift ids or don't know which shifts are still available, they can send **REQUEST**, which will return all available shifts. Employees who request shifts, will receive an SMS informing them how many have been successfully secured, how may are already taken and which do not exist. When a shift is successfully booked, its status will be updated to **taken** in the **shift_status** DynamoDB table.

Employess who send an SMS with invalid content, they will receive an SMS informing them that the SMS sent was invalid and that they can send **REQUEST** to retrieve all available shifts.

This solution uses a "dummy" **whitelist** with the employee numbers that can access this service. The mobile numbers of all senders are being checked against the whitelist and if they are not part of it they will receive an SMS saying, access denied. You should maintain a **whitelist** in a separate DB if you are planning to impelement this solution in production.

### Inbound SMS logic
![inbound-sms-logic](https://github.com/Pioank/shift-notification-and-booking-sms/blob/main/assets/inbound-sms-b-logic.PNG)

### Inbound SMS architecture
![inbound-sms-architecture](https://github.com/Pioank/shift-notification-and-booking-sms/blob/main/assets/inbound-sms-arch.PNG)

## Implementation

This solution can be deployed using AWS CloudFormation.

### Prerequisites
1. Have an originating identity that supports two-way SMS
2. Have a Pinpoint project, all SMS endpoints imported and a dynamic segment with all employess interested in extra shifts

### Implementation steps
1. Download the [CloudFormation template](https://github.com/Pioank/shift-notification-and-booking-sms/blob/main/CF-shift-notification-and-booking-sms.yaml) and deploy the solution in the same AWS region as your Pinpoint project
2. Fill the CloudFormation parameters as shown below:
    1. **ApprovedNumbers**: Type all mobile numbers that are allowed to use this serivce. The format should be E164 +<country-code><number> and they should be separated by commas e.g. +4457434243,+432434324. This field is for demo purposes and you should update the AWS Lambda code to query a DB that contains all whitelisted employee mobile numbers.
    2. **OriginationNumber**: Type the mobile number that you have in your Amazon Pinpoint account in E164 format E164 +<country-code><number> e.g. +44384238975
    3. **PinpointProjectId**: Type the existing Amazon Pinpoint project ID
    4. **SegmentId**: Type the Amazon Pinpoint existing dynamic segment ID that you would like to send the SMS notifications to
    5. **ConnectEnable**: Select **YES** if you have already have a Connect instance with a Contact Flow and Queue. If you select **NO** ignore all the fields below, the solution will still be deployed but employees won't be able to request a call
    6. **InstanceId**: Type the Connect InstanceId
    7. **ContactFlowID**: Type the Connect Contact Flow Id that you want this solution to use
    8. **QueueID**: Type the Queue Id
    9. **SourcePhoneNumber**: Type the Connect number in E164 format that is connected to the Contact Flow provided in step 7
    
3. Once the solution has been successfully deployed, navigate to the DynamoDB console and access the **ShiftsStatusDynamoDB** table and create as many items as new shifts needed. Each item should have a **shift_id** that employees will use to book the shifts, a column **shift_status** with the value **available** and a column **shift_info** where you can put additional information about the shift. See example below: ![shift_status_db](https://github.com/Pioank/shift-notification-and-booking-sms/blob/main/assets/shift_status_dynamoDB.PNG)
4. Navigate to **Amazon Pinpoint console > SMS and voice > Phone numbers**, select the **Number** that you used as **OriginationNumber** in the CloudFormation above, enable **Two-way SMS**, under the **Incoming messages destination** select **Choose an existing SNS topic** and select the one containing the name **TwoWaySMSSNSTopic**
5. Navigate to the DynamoDB console and access the **ShiftsCampaignDynamoDB** table. Each item you create represents an Amazon Pinpoint SMS campaign. Create an item and provide a **campaign_id**, which will be used as the Amazon Pinpoint Campaign name. Create a new attribute (string) with the name **shifts** and type all shifts available that you want to communicate via this campaign. It is important to contain the shift id for each of them so employees can request them. See example below:
**Note:** By completing Step 5 above, you will trigger an Amazon Pinpoint SMS Campaign. You access the campaign information and analytics from the Amazon Pinpoint console.
![shift_campaign_db](https://github.com/Pioank/shift-notification-and-booking-sms/blob/main/assets/shift_campaign_dynamoDB.PNG)

## Testing
1. Make sure you have created the shifts in the **ShiftsStatusDynamoDB** table
2. To test the SMS Campaign, replicate Step 5 under **Implementation**
3. Send SMS to your application mobile number to test the responses:
    1. Send REQUEST to receive all shifts with shift_status = available
    2. Send a shift_id that doesn't exist to receive an automatic response "This is not a valid shift code, please reply by typing REQUEST to view the available shifts"
    3. Send a valid & available shift_id to book the shift and then check the **ShiftsStatusDynamoDB** table to see that the shift_status is now **taken** and there is a new column **employee** with the mobile number of the employee who has requested it
    4. If have deployed the solution along with Amazon Connect, send AGENT and await for the call
