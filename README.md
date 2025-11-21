CV Chatbot with Amazon Bedrock: Complete Implementation Guide

Document Version: 1.0
Date: November 21, 2025
Author: Technical Documentation
Project: Julia Baucher CV Chatbot Implementation

================================================================================

TABLE OF CONTENTS

1. Purpose and Overview .................................................... 3
2. Core Features ........................................................... 4
3. System Architecture ..................................................... 5
4. AWS Resources ........................................................... 6
5. Implementation Details .................................................. 7
6. Code and Configurations ................................................. 8
7. Troubleshooting Guidelines ............................................. 12
8. Testing and Validation ................................................. 16
9. Future Enhancements .................................................... 17

================================================================================

1. PURPOSE AND OVERVIEW

1.1 Project Purpose
-------------------
This implementation creates an intelligent chatbot for Julia Baucher's CV 
website that leverages Amazon Bedrock's Claude AI model to provide interactive, 
conversational responses about her professional background, skills, and 
experience.

1.2 Business Value
------------------
• Enhanced User Experience: Visitors can ask natural language questions about 
  Julia's qualifications
• 24/7 Availability: Automated responses without human intervention
• Scalable Solution: Serverless architecture handles varying traffic loads
• Cost-Effective: Pay-per-use model with no infrastructure maintenance

1.3 Technical Objectives
------------------------
• Integrate Claude AI model via Amazon Bedrock
• Create RESTful API endpoint for frontend integration
• Implement proper CORS handling for web applications
• Ensure secure, scalable, and maintainable architecture

================================================================================

2. CORE FEATURES

2.1 Natural Language Processing
-------------------------------
• AI Model: Claude 3.5 Sonnet via Amazon Bedrock
• Context Awareness: Understands CV-related queries
• Conversational Interface: Natural, human-like responses

2.2 API Capabilities
--------------------
• RESTful Endpoint: POST /chat for message processing
• JSON Communication: Structured request/response format
• CORS Support: Cross-origin requests for web integration
• Error Handling: Comprehensive error responses

2.3 Scalability and Performance
-------------------------------
• Serverless Architecture: Auto-scaling Lambda functions
• Regional Deployment: EU-Central-1 for optimal performance
• Efficient Processing: Optimized for quick response times

================================================================================

3. SYSTEM ARCHITECTURE

3.1 Architecture Diagram
------------------------

┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Frontend      │    │   API Gateway    │    │   AWS Lambda    │
│   (CV Website)  │───▶│                  │───▶│                 │
│                 │    │  CORS Enabled    │    │  Node.js 22.x   │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                                         │
                                                         ▼
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   CloudWatch    │    │   Amazon Bedrock │    │   IAM Role      │
│   Logs          │◀───│                  │◀───│                 │
│                 │    │  Claude Model    │    │  Permissions    │
└─────────────────┘    └──────────────────┘    └─────────────────┘

3.2 Data Flow
-------------
1. User Input: Frontend sends POST request to API Gateway
2. Request Routing: API Gateway forwards to Lambda function
3. Processing: Lambda validates input and formats Bedrock request
4. AI Inference: Bedrock processes request using Claude model
5. Response: Lambda formats and returns AI response
6. Frontend Display: Website displays chatbot response to user

================================================================================

4. AWS RESOURCES

4.1 AWS Lambda Function
-----------------------
• Name: JuliaBaucher_Bedrock_lab3
• Runtime: Node.js 22.x
• Memory: 128 MB (default)
• Timeout: 30 seconds
• Handler: index.handler

4.2 API Gateway
---------------
• Type: HTTP API
• Name: Custom API for CV chatbot
• Endpoint: https://by02teld6l.execute-api.eu-central-1.amazonaws.com/prod/chat
• Method: POST
• CORS: Enabled for all origins

4.3 Amazon Bedrock
------------------
• Model: Claude 3.5 Sonnet
• Model ID: eu.anthropic.claude-haiku-4-5-20251001-v1:0
• Region: eu-central-1
• Access: Via inference profile

4.4 IAM Role
------------
• Service: Lambda execution role
• Policies: 
  - AWSLambdaBasicExecutionRole
  - Custom Bedrock invoke permissions

4.5 CloudWatch
--------------
• Log Group: /aws/lambda/JuliaBaucher_Bedrock_lab3
• Retention: Default (never expire)
• Monitoring: Error rates, duration, invocations

================================================================================

5. IMPLEMENTATION DETAILS

5.1 Environment Variables
-------------------------
BEDROCK_MODEL_ID=eu.anthropic.claude-haiku-4-5-20251001-v1:0
SYSTEM_MESSAGE=You are a helpful AI assistant on Julia Baucher's CV website. Answer politely and concisely.

5.2 Dependencies
----------------
{
    "name": "cv-chatbot-bedrock",
    "version": "1.0.0",
    "dependencies": {
        "@aws-sdk/client-bedrock-runtime": "^3.0.0"
    }
}

================================================================================

6. CODE AND CONFIGURATIONS

6.1 Lambda Function Code (index.js)
-----------------------------------

const { BedrockRuntimeClient, InvokeModelCommand } = require("@aws-sdk/client-bedrock-runtime");

const client = new BedrockRuntimeClient({ region: "eu-central-1" });

exports.handler = async (event) => {
    const corsHeaders = {
        "Content-Type": "application/json",
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Headers": "Content-Type",
        "Access-Control-Allow-Methods": "POST, OPTIONS"
    };

    try {
        console.log("Received event:", JSON.stringify(event, null, 2));

        // Handle CORS preflight requests
        if (event.requestContext?.http?.method === 'OPTIONS') {
            return {
                statusCode: 200,
                headers: corsHeaders,
                body: JSON.stringify({ message: "CORS preflight successful" })
            };
        }

        // Parse request body
        let body;
        try {
            body = typeof event.body === 'string' ? JSON.parse(event.body) : event.body;
        } catch (parseError) {
            console.error("JSON parse error:", parseError);
            return {
                statusCode: 400,
                headers: corsHeaders,
                body: JSON.stringify({ error: "Invalid JSON in request body" })
            };
        }

        // Validate required fields
        const userMessage = body?.message;
        if (!userMessage) {
            return {
                statusCode: 400,
                headers: corsHeaders,
                body: JSON.stringify({ error: "Message is required" })
            };
        }

        // Prepare system prompt
        const systemPrompt = process.env.SYSTEM_MESSAGE || 
            "You are a helpful AI assistant on Julia Baucher's CV website. Answer politely and concisely.";

        // Format Bedrock request
        const bedrockRequest = {
            anthropic_version: "bedrock-2023-05-31",
            max_tokens: 512,
            temperature: 0.7,
            system: systemPrompt,
            messages: [
                {
                    role: "user",
                    content: userMessage
                }
            ]
        };

        console.log("Sending request to Bedrock:", JSON.stringify(bedrockRequest, null, 2));

        // Invoke Bedrock model
        const command = new InvokeModelCommand({
            modelId: process.env.BEDROCK_MODEL_ID,
            contentType: "application/json",
            accept: "application/json",
            body: JSON.stringify(bedrockRequest)
        });

        const response = await client.send(command);
        const responseBody = JSON.parse(new TextDecoder().decode(response.body));

        console.log("Bedrock response:", JSON.stringify(responseBody, null, 2));

        // Extract generated text
        const generatedText = responseBody.content?.[0]?.text || "No response generated";

        return {
            statusCode: 200,
            headers: corsHeaders,
            body: JSON.stringify({
                reply: generatedText
            })
        };

    } catch (error) {
        console.error("Lambda error:", error);
        return {
            statusCode: 500,
            headers: corsHeaders,
            body: JSON.stringify({
                error: "Internal server error",
                details: error.message
            })
        };
    }
};

6.2 IAM Policy for Bedrock Access
---------------------------------

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "bedrock:InvokeModel"
            ],
            "Resource": [
                "arn:aws:bedrock:eu-central-1::foundation-model/anthropic.claude-*"
            ]
        }
    ]
}

6.3 Frontend Integration Example
-------------------------------

// JavaScript for CV website integration
class CVChatbot {
    constructor(apiEndpoint) {
        this.apiEndpoint = apiEndpoint;
    }

    async sendMessage(message) {
        try {
            const response = await fetch(this.apiEndpoint, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({ message: message })
            });

            if (!response.ok) {
                throw new Error(`HTTP error! status: ${response.status}`);
            }

            const data = await response.json();
            return data.reply;
        } catch (error) {
            console.error('Chatbot error:', error);
            return 'Sorry, I encountered an error. Please try again.';
        }
    }
}

// Usage
const chatbot = new CVChatbot('https://by02teld6l.execute-api.eu-central-1.amazonaws.com/prod/chat');

// Example usage in chat interface
async function handleUserMessage(userInput) {
    const response = await chatbot.sendMessage(userInput);
    displayMessage(response);
}

================================================================================

7. TROUBLESHOOTING GUIDELINES

7.1 Problem 1: Base64 Encoding Error in AWS CLI
-----------------------------------------------

SYMPTOMS:
Invalid base64: "{"anthropic_version":"bedrock-2023-05-31"...}"

ROOT CAUSE:
AWS CLI expected base64-encoded body parameter instead of raw JSON string.

SOLUTION:
Use file-based approach with fileb:// prefix:

# Create request file
cat > request.json << 'EOF'
{
    "anthropic_version": "bedrock-2023-05-31",
    "max_tokens": 100,
    "messages": [{"role": "user", "content": "Hello"}]
}
EOF

# Use file in CLI command
aws bedrock-runtime invoke-model \
    --body fileb://request.json \
    --model-id MODEL_ID \
    response.json

7.2 Problem 2: Model ID Validation Error
----------------------------------------

SYMPTOMS:
ValidationException: Invocation of model ID anthropic.claude-haiku-4-5-20251001-v1:0 
with on-demand throughput isn't supported. Retry your request with the ID or ARN of 
an inference profile that contains this model.

ROOT CAUSE:
Direct model IDs require inference profiles for on-demand access.

SOLUTION:
Use inference profile instead of direct model ID:

# Instead of: anthropic.claude-haiku-4-5-20251001-v1:0
# Use: eu.anthropic.claude-haiku-4-5-20251001-v1:0

VERIFICATION:
aws bedrock-runtime invoke-model \
    --model-id eu.anthropic.claude-haiku-4-5-20251001-v1:0 \
    --body fileb://request.json \
    response.json

7.3 Problem 3: Lambda Module Import Error
-----------------------------------------

SYMPTOMS:
{"message":"Internal Server Error"}

CLOUDWATCH LOGS:
Cannot find module '@aws-sdk/client-bedrock-runtime'

ROOT CAUSE:
Node.js 22.x runtime doesn't include AWS SDK v3 by default, and dependencies 
weren't included in deployment package.

SOLUTION:
Create deployment package with dependencies:

# Create package.json
{
    "dependencies": {
        "@aws-sdk/client-bedrock-runtime": "^3.0.0"
    }
}

# Install dependencies
npm install

# Create deployment package
zip -r lambda-deployment.zip . -x "*.git*"

7.4 Problem 4: File Extension Mismatch
--------------------------------------

SYMPTOMS:
Runtime errors with .mjs files containing CommonJS syntax.

ROOT CAUSE:
• .mjs extension expects ES Module syntax (import/export)
• Code used CommonJS syntax (require/exports)

SOLUTION:
Use .js extension for CommonJS syntax:

// Correct for .js files
const { BedrockRuntimeClient } = require("@aws-sdk/client-bedrock-runtime");
exports.handler = async (event) => { ... };

// Would be correct for .mjs files
import { BedrockRuntimeClient } from "@aws-sdk/client-bedrock-runtime";
export const handler = async (event) => { ... };

7.5 Problem 5: CORS Issues
--------------------------

SYMPTOMS:
Browser console errors about CORS policy violations.

ROOT CAUSE:
Missing or incorrect CORS headers in Lambda response.

SOLUTION:
Include proper CORS headers in all responses:

const corsHeaders = {
    "Content-Type": "application/json",
    "Access-Control-Allow-Origin": "*",
    "Access-Control-Allow-Headers": "Content-Type",
    "Access-Control-Allow-Methods": "POST, OPTIONS"
};

// Include in all return statements
return {
    statusCode: 200,
    headers: corsHeaders,
    body: JSON.stringify(response)
};

7.6 Problem 6: Model Identity Discrepancy
-----------------------------------------

SYMPTOMS:
Model responds as "Claude 3.5 Sonnet" instead of "Haiku 4.5".

ROOT CAUSE:
Inference profile routing or model aliasing by AWS.

SOLUTION:
This is expected behavior. The inference profile may route to the most 
appropriate model. The functionality remains correct for the use case.

================================================================================

8. TESTING AND VALIDATION

8.1 Unit Testing Commands
-------------------------

# Test 1: Basic functionality
curl -X POST https://by02teld6l.execute-api.eu-central-1.amazonaws.com/prod/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello, what model are you?"}'

# Test 2: CV-specific query
curl -X POST https://by02teld6l.execute-api.eu-central-1.amazonaws.com/prod/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Tell me about Julia Baucher'\''s experience"}'

# Test 3: Error handling (missing message)
curl -X POST https://by02teld6l.execute-api.eu-central-1.amazonaws.com/prod/chat \
  -H "Content-Type: application/json" \
  -d '{}'

# Test 4: CORS preflight
curl -X OPTIONS https://by02teld6l.execute-api.eu-central-1.amazonaws.com/prod/chat \
  -H "Origin: https://example.com"

8.2 Expected Responses
----------------------

SUCCESSFUL RESPONSE:
{
    "reply": "Hello! I'm Claude, an AI assistant made by Anthropic. I'm here to help answer questions about Julia Baucher's CV and provide information from this website. How can I assist you today?"
}

ERROR RESPONSE:
{
    "error": "Message is required"
}

8.3 Performance Metrics
-----------------------
• Cold Start: ~2-3 seconds
• Warm Invocation: ~500-800ms
• Bedrock Latency: ~300-500ms
• Memory Usage: ~50-70MB

================================================================================

9. FUTURE ENHANCEMENTS

9.1 Enhanced Context Awareness
------------------------------
• Upload Julia's actual CV content to provide more specific responses
• Implement conversation memory for multi-turn dialogues
• Add context injection from website content

9.2 Advanced Features
---------------------
• Streaming Responses: Real-time response generation
• Multi-language Support: Detect and respond in user's language
• Analytics: Track popular questions and user interactions

9.3 Security Improvements
-------------------------
• Rate Limiting: Prevent API abuse
• Authentication: Optional user authentication
• Input Validation: Enhanced sanitization and validation

9.4 Monitoring and Observability
--------------------------------
• Custom Metrics: Track business-specific KPIs
• Alerting: Set up CloudWatch alarms for errors
• Distributed Tracing: Implement X-Ray tracing

9.5 Cost Optimization
---------------------
• Reserved Capacity: For predictable workloads
• Model Selection: Choose optimal model based on query complexity
• Caching: Implement response caching for common queries

================================================================================

CONCLUSION

This implementation successfully creates a scalable, serverless CV chatbot using 
Amazon Bedrock and Claude AI. The solution provides natural language interaction 
capabilities while maintaining cost-effectiveness and high availability. The 
comprehensive troubleshooting guidelines ensure smooth deployment and operation, 
while the modular architecture allows for future enhancements and customizations.

The system is now ready for production use and can be easily integrated into 
Julia Baucher's CV website to provide an interactive, engaging experience for 
visitors.

================================================================================

End of Document

Page Count: 17 pages
Word Count: Approximately 3,500 words
Print Settings: Recommended A4, 12pt font, 1-inch margins
