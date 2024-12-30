<h1>Student Registration System </h1>

The student profile is existing. The student needs to register for the exam at a specific centre at a specific date.
So we have four types of requests coming from the upstream application into our SNS (notification service)

1. An SNS Topic is created with filter policies in place for the 4 business rules. i.e. the 4 type of requests:
   Registration(Bulk / single), Cancellation (Bulk / single)
   so depending on the JSON payload(type of request), it will be sent to the SQS Queues which have subscribed to the SNS Topic.

2. we have <b>4 SQS queues </b> corresponding to the four types of payloads coming from SNS. Each Queue has a correspondind DLQ in case of any failure.
   We need a queue to retain the information coming from the SNS, as the SNS does have the ability to retain the information(students information and the payment details)

3. the queue triggers the <b>Lambda</b> which was created with this sqs as the source.
   Lambda validation(missing data validation, single or bulk)success will result in an ackowledgement. else the request is retained and tried for upto 3 times before sendig to the DLQ.

4. DLQ contains a retry logic . If request still fails, we max out. These failed(Maxed Out) requests are processed manually and analysed for missing data, and any incomplete imformation is updated and the request is resent with new data.

5. Successful Lambda calls the required <b>SF </b>
   SF has the 3 <b>Lambdas</b>

   1. validate the payload i.e the student attribute are validated for required fields,
   2. do next level processing (an insertion in the DynamoDB registration table), registration processing, which result in getting a Registration ID. along with success acknowlegdement sent to the team.
   3. the student master table is updated with registration ID

6. This registration ID along with the initial payment details are sent to the downstream application for payment processing.

7. all payloads that are successfully registered are also converted to messages and stored in S3 bucket. each student(studentID , RegistrationTime) is represented by one JSON.

8. there is a sheduled job which is running at a cutoff time morning and evening , which takes all the S3 Data, writes an Athena query. converts all the JSON records into one single CSV file , then this csv file is converted to the required format, as requested by the downstream application, which we place in a <b>S3 location (or FTP location)</b> with the help of assume role and Authorisation Token. So we create an archive file and delete all the data from our S3 Location.

9. Athena has query option to generate CSV by using Lambda eg. select \* from collection-name, so we can get all the result records into a CSV file directly.
