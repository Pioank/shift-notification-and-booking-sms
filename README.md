# shift-notification-and-booking-sms

## Description

Companies might have extra shifts for their employees, which are considered overtime and are not allocated. These shifts are usually communicated in advance and employees who are seeking to generate extra income can book them. There are two parts in this process:

- The company communicating the available shifts to employees who are interested
- Employees confirming, which of the available shifts want

Such use cases involve employees who hold manual posts with little access to the internet. SMS is the preffered channel to communicate available shifts as well as receive the employees' requests. Using SMS ensures that everyone can receive and book available shifts with no accessibility issues. 

## Solution

The solution in this repository covers both parts of the process described above.

**Communicating available shifts to employees who are interested:**
New shifts should be recorded in a data base, along with their status (available / taken) and other information that can be useful when sending the message. This solution uses DynamoDB to store the shifts with the **key = shift_id**. When new shifts are added, marketers or an automated process, will create a new line in a second DynamoDB table with **key = campaign_id**. This DynamoDB table comprises from the **key** and a free text field containing a list of the available shifts. Users can add any information about the shift they want to as long as they contain the **shift_id**.

Whenever a new line is created in the **campaign_id** DynamoDB table, DynamoDB-Streams invokes an AWS Lambda function that creates an Amazon Pinpoint SMS Campaign using a dynamic segment that includes all employees interested in extra shifts. Note that it is assumed you have already imported your employee data as Pinpoint SMS endpoints and have created a dynamic semgent that includes all employees interested in extra shifts.

The message sent to the employees contains 

**Employees confirming, which of the available shifts want**
Once the employees


