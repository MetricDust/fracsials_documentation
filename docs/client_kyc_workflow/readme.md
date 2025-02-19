# AUTO KYC 

## Resources used

* AWS S3 Bucket (to store KYC documents)
* AWS DynamoDB Table (to store KYC user data) 
  * Table name: {stage}_kyc_service
  * Table Key: 
    * Partition Key: tenant#email
    * Sort Key: id_type#id
  * Currently, No Indexes.
* AWS Lambda Function
  * Lambda Function to save KYC data to DynamoDB
  * Lambda Function to validate KYC data (attached to DynamoDB Table stream)
  * Lambda Function validate kyc.
  * Lambda Function to send notification to user
  * Lambda Function to manually validate KYC data
  * Lambda Function to update Iam data
* AWS API Gateway to trigger Lambda Function to save KYC data
* AWS SNS Topic to send notification to user

## Architecture

![image](./kyc-architecture.png)

## Steps

### 1. User uploads KYC documents to S3 bucket and gets document url.
When a user uploads KYC documents, they receive a URL for accessing the documents. Here is an example response from the S3 upload API:

Example Response from S3 upload API:

```json
{
   "filename": "MicrosoftTeams-image (6).png",
   "url": "https://cxd2blbkp1.execute-api.us-east-1.amazonaws.com/beta/get_document_link/tenant/ashish/large_file/root/document-manager-storage/source_path/YXNoaXNoL3ByaXZhdGUvZGFzaGJvYXJkL2t5Y19kb2N1bWVudHMvOGUxNDExNWRkYTliOGZhZGJhOGFiNmQ5YWFkOWNkOGQv/1713597964_MicrosoftTeams-image-6-.png"
}
```

### 2. API Gateway triggers Lambda Function to save KYC data to DynamoDB.
Once the documents are uploaded, the API Gateway triggers a Lambda function to save the KYC data to DynamoDB. The data is sent in the following format:

```bash
GET kyc/stage/{stage}/tenant/{tenant}/create_or_update
```

```json

{
 "date_of_birth": "2024-04-20",
 "document_links": [
  {
   "filename": "MicrosoftTeams-image (6).png",
   "url": "https://cxd2blbkp1.execute-api.us-east-1.amazonaws.com/beta/get_document_link/tenant/ashish/large_file/root/document-manager-storage/source_path/YXNoaXNoL3ByaXZhdGUvZGFzaGJvYXJkL2t5Y19kb2N1bWVudHMvOGUxNDExNWRkYTliOGZhZGJhOGFiNmQ5YWFkOWNkOGQv/1713597964_MicrosoftTeams-image-6-.png"
  },
  {
   "filename": "MicrosoftTeams-image (6).png",
   "url": "https://cxd2blbkp1.execute-api.us-east-1.amazonaws.com/beta/get_document_link/tenant/ashish/large_file/root/document-manager-storage/source_path/YXNoaXNoL3ByaXZhdGUvZGFzaGJvYXJkL2t5Y19kb2N1bWVudHMvOGUxNDExNWRkYTliOGZhZGJhOGFiNmQ5YWFkOWNkOGQv/1713597964_MicrosoftTeams-image-6-.png"
  }
 ],
 "email": "ashish.p@metricdust.com",
 "first_name": "Ashish",
 "id_type": "Driving license",
 "id_value": "1234",
 "last_name": "Palankar",
 "stage": "beta",
 "tenant": "ashish"
}
```

Action:

- API Gateway invokes the Lambda function to save KYC data to DynamoDB.
- The Lambda function creates a JSON object containing KYC data (document URL, user details, etc.).
- Retrieves the user's Email and Phone Number verification status from Amazon Cognito, updates JSON object accordingly.
- It sets the `verification_status` to `pending` and `verify` as `True` in the DynamoDB table, updating the item with the JSON object.

Example Json Object:
```json
{
 "tenant#email": "ashish#ashish.p@metricdust.com",
 "id_type#id": "Driving license#1234",
 "date_of_birth": "2024-04-20",
 "document_links": [
  {
   "filename": "MicrosoftTeams-image (6).png",
   "url": "https://cxd2blbkp1.execute-api.us-east-1.amazonaws.com/beta/get_document_link/tenant/ashish/large_file/root/document-manager-storage/source_path/YXNoaXNoL3ByaXZhdGUvZGFzaGJvYXJkL2t5Y19kb2N1bWVudHMvOGUxNDExNWRkYTliOGZhZGJhOGFiNmQ5YWFkOWNkOGQv/1713597964_MicrosoftTeams-image-6-.png"
  },
  {
   "filename": "MicrosoftTeams-image (6).png",
   "url": "https://cxd2blbkp1.execute-api.us-east-1.amazonaws.com/beta/get_document_link/tenant/ashish/large_file/root/document-manager-storage/source_path/YXNoaXNoL3ByaXZhdGUvZGFzaGJvYXJkL2t5Y19kb2N1bWVudHMvOGUxNDExNWRkYTliOGZhZGJhOGFiNmQ5YWFkOWNkOGQv/1713597964_MicrosoftTeams-image-6-.png"
  }
 ],
 "email": "ashish.p@metricdust.com",
 "first_name": "Ashish",
 "id_type": "Driving license",
 "id_value": "1234",
 "last_name": "Palankar",
 "last_updated_time": 1713597971, 
 "stage": "beta",
 "tenant": "ashish",
 "validation_status": "pending",
 "verify": true
}
```


### 3. Stream from DynamoDB triggers Lambda Function to validate KYC data
Once the KYC data is saved, a DynamoDB stream triggers another Lambda function to validate the data.

Action:

- The Lambda function checks if the verify field is true.
  - If true, it performs validation using:
  - AWS Rekognition: For validating selfie images.
  - AWS Textract: For validating documents.
  - It uses the output from Rekognition and Textract, along with an AI model, to validate the KYC data.
  - if validation is successful, set `validation_status` to `verified`
  - if validation is not successful, set `validation_status` to `rejected`
- The result of the validation is stored back in DynamoDB, and the `verify` field is updated to `false` to avoid re-validation.

### 4. Lambda Function to update IdentityManagement table
This Lambda function updates the identity management table based on KYC verification results.

Action:

- Check `verity` field is `False` then send notification, else do nothing
- If KYC data is validated as approved, the status is updated to verified.
- If rejected, the status is set to rejected with the rejection reason.

### 5. Lambda Function to send notification to sns subscription
Once the KYC data is validated, a Lambda function sends a notification via SNS to the user.

Action:

- Check `verity` field is `False` then send notification, else do nothing
- Once the KYC data is validated and the final status is updated in DynamoDB, a Lambda function sends a notification via SNS to the user.
- Notifications include approval/rejection status and any next steps or reasons for rejection.

### 6. API Gateway to trigger Lambda Function to manually validate KYC data
In case manual validation is needed, an admin can trigger this function to bypass automatic verification.

```bash
POST kyc/stage/{stage}/tenant/{tenant}/email/{email}/validate
```
```json
{
  "tenant": "ashish",
  "stage": "beta",
  "email": "ashish.p@metricdust.com",
  "id_type": "Driving license",
  "id_value": "1234",
  "verification_status": "verified"
}
```
Action:

- If necessary, an `admin` can manually validate KYC data, bypassing the automatic verification steps.
- The manual validation would update the status of the KYC record in DynamoDB.

### Api Gateway to get KYC data
API Gateway triggers a Lambda function to get KYC data from DynamoDB.
```bash
GET kyc/stage/{stage}/tenant/{tenant}/email/{email}
```

Action:

- API Gateway triggers a Lambda function to get KYC data from DynamoDB.
- The Lambda function retrieves KYC data based on the provided tenant and email.