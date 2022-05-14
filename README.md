# shift-notification-and-booking-sms

## Description

Companies might have extra shifts for their employees, which are considered overtime and are not allocated. These shifts are usually communicated in advance and employees who are seeking to generate extra income can book them. There are two parts in this process:

- The company communicating the available shifts to employees who are interested
- Employees confirming, which of the available shifts want

Such use cases involve employees who hold manual posts with little access to the internet. SMS is the preffered channel to communicate available shifts as well as receive the employees' requests. Using SMS ensures that everyone can receive and book available shifts with no accessibility issues. 

## Solution

The solution in this repository covers both parts of the process described above.

**Communicating available shifts to employees who are interested:**
New shifts should be recorded in a data base, along with their status (available / taken) and other information that can be useful when sending the message. This solution uses a DynamoDB table called **shift_status** to store the shifts with the **key = shift_id**. When new shifts are added, marketers or an automated process, will create a new line in a second DynamoDB table called **shift_campaigns**  with **key = campaign_id**. The **shift_campaigns** DynamoDB table comprises from the **key** and a free text field containing a list of the available shifts. Users can add any information about the shift they want to as long as they contain the **shift_id**.

Whenever a new line is created in the **shift_campaigns** DynamoDB table, DynamoDB-Streams invokes an AWS Lambda function that creates an Amazon Pinpoint SMS Campaign using a dynamic segment, which includes all employees interested in extra shifts. Note that it is assumed you have already imported your employee data as Pinpoint SMS endpoints and have created a dynamic semgent that includes all employees interested in extra shifts.

The message sent to the employees contains the **shifts**, which are being retrieved from the **shifts_campaigns** DynamoDB table and a static part to inform recipients how to reply. See example below:
"Available shifts: ' + **shifts** + '. To secure a shift reply by typing the codes separated by comma e.g. XYZ123, XYZ222. You can always text REQUEST to see which shifts are still available or text STOP to stop receiving such notifications"

### SMS campaign logic
![sms-campaign-logic](https://github.com/Pioank/shift-notification-and-booking-sms/blob/main/assets/sms-campaign-logic.PNG)

### SMS campaign architecture
![sms-campaign-architecture](https://github.com/Pioank/shift-notification-and-booking-sms/blob/main/assets/sms-campaign-architecture.PNG)

**Employees confirming, which of the available shifts want**
Employees can confirm, which shifts want to sign-up for by sending an SMS with the shift ids separated by comma. If they cannot remember the shift ids or don't know which shifts are still available, they can send **REQUEST**, which will return all available shifts. Employees who request shifts, will receive an SMS informing them how many have been successfully secured, how may are already taken and which do not exist. When a shift is successfully booked, its status will be updated to **taken** in the **shift_status** DynamoDB table.

Employess who send an SMS with invalid content, they will receive an SMS informing them that the SMS sent was invalid and that they can send **REQUEST** to retrieve all available shifts.

This solution uses a "dummy" **whitelist** with the employee numbers that can access this service. The mobile numbers of all senders are being checked against the whitelist and if they are not part of it they will receive an SMS saying, access denied. You should maintain a **whitelist** in a separate DB if you are planning to impelement this solution in production.

### Inbound SMS logic
![inbound-sms-logic](https://github.com/Pioank/shift-notification-and-booking-sms/blob/main/assets/inbound-sms-logic.PNG)

### Inbound SMS architecture
![inbound-sms-architecture](https://github.com/Pioank/shift-notification-and-booking-sms/blob/main/assets/inbound-sms-architecture.PNG)

## Implementation

This solution can be deployed using AWS CloudFormation.

### Prerequisites
1. Have an originating identity that supports two-way SMS
2. Have a Pinpoint project, all SMS endpoints imported and a dynamic segment with all employess interested in extra shifts

### Implementation steps
1. Deploy the solution in the same AWS region as your Pinpoint project
2. 
