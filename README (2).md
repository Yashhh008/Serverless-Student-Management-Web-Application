# Serverless Student Management Web Application

A serverless academic project for managing student records using **AWS
Lambda, Amazon API Gateway, and Amazon DynamoDB**. The frontend uses
HTML, CSS, and vanilla JavaScript.

Users can add students, view all records, search by student ID, and
delete records. Student data is stored persistently in DynamoDB.

## Table of Contents

-   [Features](#features)
-   [Architecture](#architecture)
-   [Technology Stack](#technology-stack)
-   [Project Structure](#project-structure)
-   [API Endpoints](#api-endpoints)
-   [Database Design](#database-design)
-   [AWS Setup and Deployment](#aws-setup-and-deployment)
-   [Run the Frontend Locally](#run-the-frontend-locally)
-   [Testing](#testing)
-   [Troubleshooting](#troubleshooting)
-   [Limitations and Future
    Enhancements](#limitations-and-future-enhancements)
-   [Cost and Cleanup](#cost-and-cleanup)

## Features

-   Create a student record with ID, name, email, and course.
-   List all stored students.
-   Search for a student by ID.
-   Delete a student by ID.
-   Persist records in DynamoDB across page refreshes.
-   Use a Python Lambda function as the backend.
-   Expose backend operations through API Gateway HTTP routes.
-   Validate required fields and reject duplicate student IDs.
-   Responsive HTML/CSS/JavaScript frontend.

## Architecture

``` mermaid
flowchart TD
    U[User / Browser] --> F[HTML CSS JavaScript Frontend]
    F -->|HTTP requests| A[Amazon API Gateway HTTP API]
    A -->|Invoke| L[AWS Lambda - Python]
    L -->|Read and write| D[(DynamoDB Students Table)]
    I[IAM Execution Role] -.->|Authorizes access| L
    L -.->|Execution logs| C[Amazon CloudWatch Logs]
    A -->|HTTP response| F
```

### Component responsibilities

  -----------------------------------------------------------------------
  Component                           Responsibility
  ----------------------------------- -----------------------------------
  HTML, CSS, JavaScript               Renders the UI and sends HTTP
                                      requests.

  Amazon API Gateway                  Exposes HTTP routes and forwards
                                      requests to Lambda.

  AWS Lambda                          Validates input, handles routing,
                                      and executes database operations.

  Amazon DynamoDB                     Stores student records
                                      persistently.

  AWS IAM                             Restricts the AWS operations
                                      available to the Lambda execution
                                      role.

  Amazon CloudWatch Logs              Stores Lambda execution logs for
                                      troubleshooting.
  -----------------------------------------------------------------------

### Request flow: create a student

1.  The user submits the student form.
2.  The frontend sends `POST /students` with a JSON body.
3.  API Gateway invokes the Lambda function.
4.  Lambda validates required fields and prevents duplicate student IDs.
5.  Lambda writes the student item to DynamoDB.
6.  Lambda returns a response through API Gateway.
7.  The frontend reloads the student list.

## Technology Stack

-   **Frontend:** HTML5, CSS3, JavaScript
-   **Backend:** Python
-   **Compute:** AWS Lambda
-   **API:** Amazon API Gateway HTTP API
-   **Database:** Amazon DynamoDB
-   **Permissions:** AWS IAM
-   **Logging:** Amazon CloudWatch Logs
-   **Testing:** Browser and optionally Postman

## Project Structure

The frontend is a single HTML file. The backend code is deployed in the
Lambda console.

``` text
serverless-student-management/
├── student_management_frontend.html
└── README.md

AWS Lambda source:
└── lambda_function.py
```

The Lambda runtime includes `boto3`, which is used to access DynamoDB.

## API Endpoints

Replace `YOUR_API_INVOKE_URL` with the Invoke URL shown in API Gateway.
Do not append `/students` to the base URL itself.

  Method     Endpoint                  Purpose               Success status
  ---------- ------------------------- --------------------- ----------------
  `GET`      `/students`               List all students     `200`
  `GET`      `/students/{studentId}`   Get a student by ID   `200`
  `POST`     `/students`               Create a student      `201`
  `DELETE`   `/students/{studentId}`   Delete a student      `200`

### Create a student

``` http
POST YOUR_API_INVOKE_URL/students
Content-Type: application/json
```

Example JSON body:

``` json
{
  "studentId": "STU003",
  "name": "Asha Kumar",
  "email": "asha@example.com",
  "course": "CSE-AIML"
}
```

A successful request returns HTTP `201`. Required fields are
`studentId`, `name`, `email`, and `course`. Blank or missing fields are
rejected, and duplicate IDs are prevented.

### List all students

``` http
GET YOUR_API_INVOKE_URL/students
```

Example response:

``` json
{
  "students": [
    {
      "studentId": "STU003",
      "name": "Asha Kumar",
      "email": "asha@example.com",
      "course": "CSE-AIML"
    }
  ]
}
```

### Get one student

``` http
GET YOUR_API_INVOKE_URL/students/STU003
```

Returns the matching student or `404` if no matching record exists.

### Delete a student

``` http
DELETE YOUR_API_INVOKE_URL/students/STU003
```

Returns `200` on successful deletion or `404` if the student does not
exist.

### Common response codes

  Status   Meaning
  -------- -------------------------------------------
  `200`    Request succeeded
  `201`    Student created
  `400`    Invalid input or unsupported method/route
  `404`    Route or student not found
  `500`    Unexpected application or database error

## Database Design

**Table:** `Students`\
**Partition key:** `studentId` (String)

  Attribute     Type     Description
  ------------- -------- -------------------------------------
  `studentId`   String   Unique identifier and partition key
  `name`        String   Student name
  `email`       String   Student email
  `course`      String   Course or program

The current list operation uses DynamoDB `Scan`. This is acceptable for
a small demonstration dataset, but pagination or a more targeted
data-access design is preferable as data grows.

## AWS Setup and Deployment

The project was configured for **Asia Pacific (Mumbai), `ap-south-1`**.
Use the same region for Lambda, API Gateway, and DynamoDB.

### Prerequisites

-   AWS account with permission to create Lambda, API Gateway, DynamoDB,
    and IAM resources.
-   Modern web browser and a text editor.
-   Optional: Python installed locally to serve the frontend.
-   Optional: Postman for testing POST and DELETE requests.

### 1. Create the DynamoDB table

1.  Open the DynamoDB console.
2.  Create a table named `Students`.
3.  Set partition key to `studentId` with type **String**.
4.  For a small project, on-demand capacity is convenient.
5.  Create the table.

### 2. Create the Lambda function

1.  Open AWS Lambda and create a function named `StudentManagementAPI`.
2.  Select a supported Python runtime.
3.  Create or select an execution role with basic Lambda logging
    permissions.
4.  Add the Python handler that handles the four routes.
5.  Deploy and test the function.

### 3. Grant DynamoDB permissions

Attach a narrowly scoped IAM policy to the Lambda execution role. The
current implementation needs these actions:

-   `dynamodb:PutItem`
-   `dynamodb:GetItem`
-   `dynamodb:Scan`
-   `dynamodb:DeleteItem`

Restrict the policy resource to the ARN of the `Students` table in your
account and region. Avoid unrestricted permissions such as `dynamodb:*`
on all resources.

### 4. Create the API Gateway HTTP API

1.  Open Amazon API Gateway and create an **HTTP API** named
    `StudentManagementHTTPAPI`.
2.  Add the Lambda integration for `StudentManagementAPI`.
3.  Configure each route to use the same Lambda integration:
    -   `GET /students`
    -   `POST /students`
    -   `GET /students/{studentId}`
    -   `DELETE /students/{studentId}`
4.  Configure a stage such as `$default` and ensure the API is deployed.
    Auto-deploy is convenient for a small project.
5.  Copy the API Invoke URL.

### 5. Configure CORS for browser access

Browser requests from a frontend hosted on a different origin require
suitable CORS configuration. During local testing, this project allowed
`GET`, `POST`, `DELETE`, and `OPTIONS`, and the `Content-Type` header.
The allowed origin was configured permissively during development.

For deployment, use the exact frontend origin you intend to allow. CORS
is a browser policy; it does not authenticate users or prevent direct
API calls.

### 6. Connect the frontend

1.  Open `student_management_frontend.html` in a browser or serve it
    locally.
2.  Paste the API Gateway Invoke URL into the **API Configuration**
    field.
3.  Click **Use this API URL**.
4.  Add, search, refresh, and delete a test student.

## Run the Frontend Locally

From the directory containing `student_management_frontend.html`, run:

``` bash
python -m http.server 8000
```

Open:

``` text
http://localhost:8000/student_management_frontend.html
```

Keep the terminal running while using the app. Paste the actual API
Invoke URL into the page and click **Use this API URL**. If Python is
unavailable, use another local static-file server or a live-server
extension.

## Testing

The following functional tests were performed during development:

  Test                       Expected outcome                       Result
  -------------------------- -------------------------------------- --------
  Lambda basic test          Lambda returns a successful response   Passed
  Create through Lambda      Student stored in DynamoDB             Passed
  List through Lambda        Stored records returned                Passed
  Get by ID through Lambda   Matching item returned                 Passed
  Delete through Lambda      Item removed                           Passed
  GET through API Gateway    JSON response returned                 Passed
  Add through frontend       Student appears in table               Passed
  Search through frontend    Matching student appears               Passed
  Refresh frontend           Record remains available               Passed
  Delete through frontend    Record disappears from list            Passed

Malformed JSON, missing fields, duplicate IDs, and missing student IDs
should be tested separately before claiming those cases passed.

## Troubleshooting

  -----------------------------------------------------------------------------
  Problem                       Possible cause          What to check
  ----------------------------- ----------------------- -----------------------
  Browser CORS error            CORS origin, methods,   Verify API Gateway CORS
                                or headers do not match configuration and
                                                        deployed stage

  HTTP `403`                    Incorrect URL, stage,   Verify the Invoke URL
                                route, or API access    and stage
                                configuration           

  HTTP `404`                    Route mismatch or       Check route path and
                                student not found       student ID

  HTTP `500`                    Lambda exception or     Review Lambda test
                                database permissions    output and CloudWatch
                                                        logs

  `AccessDeniedException`       Missing IAM action or   Check the Lambda
                                incorrect table ARN     execution role

  `ResourceNotFoundException`   Table name or region    Confirm `Students`
                                mismatch                exists in the Lambda
                                                        region

  Duplicate ID rejected         ID already exists       Use another ID or
                                                        remove the existing
                                                        record

  Empty list                    No records exist or     Add a test student and
                                POST failed             refresh
  -----------------------------------------------------------------------------

## Limitations and Future Enhancements

-   **Authentication:** Add Amazon Cognito or another identity provider
    and enforce authentication at API Gateway before exposing the app to
    wider use.
-   **CORS:** Restrict allowed origins to the intended frontend domain
    when deployed.
-   **Update operation:** Add a `PUT` or `PATCH` route and a DynamoDB
    update operation.
-   **Pagination:** Replace an unpaginated scan with paginated reads for
    larger datasets.
-   **Validation:** Add stricter email validation, length limits, and
    clearer errors.
-   **Frontend hosting:** Host the static frontend using an appropriate
    static hosting solution.
-   **Monitoring:** Add useful metrics, alarms, and structured logs.
-   **Automated deployment:** Add automated tests and
    infrastructure-as-code tooling.

## Cost and Cleanup

AWS costs depend on account eligibility, usage, configuration, and
current pricing. Do not assume every resource is free. Review the
pricing for Lambda, API Gateway, DynamoDB, and CloudWatch, and set
billing alerts where available.

After the demonstration, save screenshots and project files. Delete
resources you no longer need, including the API, Lambda function,
DynamoDB table, and project-specific IAM resources when safe to do so.
Check dependencies before deleting resources and review billing for any
remaining usage.

## Conclusion

This project demonstrates a serverless web application built using AWS
managed services. API Gateway exposes HTTP routes, Lambda processes
requests, DynamoDB stores student records, and IAM controls backend
access to the database. The frontend provides a simple interface to
create, retrieve, search, and delete records without maintaining a
dedicated application server.

------------------------------------------------------------------------

**Project:** Serverless Student Management Web Application\
**AWS Region used during development:** `ap-south-1` (Asia Pacific ---
Mumbai)\
**Lambda function:** `StudentManagementAPI`\
**DynamoDB table:** `Students`\
**API Gateway HTTP API:** `StudentManagementHTTPAPI`
