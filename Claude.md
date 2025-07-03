# Claude Development Guide - NewsFlow SaaS

This document provides comprehensive guidance for building the NewsFlow serverless news report SaaS application. Use this as a reference when implementing features, debugging issues, or extending functionality.

## 🏗️ Project Structure

```
newsflow/
├── frontend/                 # Next.js frontend application
│   ├── app/                 # Next.js App Router
│   │   ├── (auth)/         # Authentication pages
│   │   ├── dashboard/      # Main dashboard
│   │   ├── api/           # API routes
│   │   └── globals.css    # Global styles
│   ├── components/        # Reusable UI components
│   ├── lib/              # Utilities and configurations
│   └── types/            # TypeScript type definitions
├── backend/              # AWS Lambda functions
│   ├── functions/       # Individual Lambda functions
│   ├── shared/          # Shared utilities and types
│   └── infrastructure/ # AWS CDK infrastructure code
├── docs/               # Additional documentation
└── tests/             # Test suites
```

## 🚀 Implementation Roadmap

### Phase 1: Core Infrastructure & Authentication

#### 1.1 Setup AWS Infrastructure (AWS CDK)

Create the CDK stack for all AWS resources:

```typescript
// backend/infrastructure/newsflow-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as events from 'aws-cdk-lib/aws-events';
import * as ses from 'aws-cdk-lib/aws-ses';

export class NewsFlowStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // DynamoDB Table with single-table design
    const table = new dynamodb.Table(this, 'NewsFlowTable', {
      tableName: 'newsflow-table',
      partitionKey: { name: 'PK', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'SK', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      pointInTimeRecovery: true,
    });

    // Lambda functions
    const apiFunction = new lambda.Function(this, 'ApiFunction', {
      runtime: lambda.Runtime.NODEJS_18_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('backend/functions/api'),
      environment: {
        TABLE_NAME: table.tableName,
      },
    });

    // API Gateway
    const api = new apigateway.RestApi(this, 'NewsFlowApi', {
      restApiName: 'NewsFlow API',
      defaultCorsPreflightOptions: {
        allowOrigins: apigateway.Cors.ALL_ORIGINS,
        allowMethods: apigateway.Cors.ALL_METHODS,
      },
    });

    // EventBridge for scheduled reports
    const eventBridge = new events.EventBus(this, 'NewsFlowEventBus');
  }
}
```

#### 1.2 Frontend Authentication Setup

```typescript
// frontend/lib/auth.ts
import NextAuth from 'next-auth';
import GoogleProvider from 'next-auth/providers/google';

export const authOptions = {
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
  ],
  callbacks: {
    async jwt({ token, account, user }) {
      if (account) {
        token.accessToken = account.access_token;
        token.userId = user.id;
      }
      return token;
    },
    async session({ session, token }) {
      session.accessToken = token.accessToken;
      session.userId = token.userId;
      return session;
    },
  },
};

export default NextAuth(authOptions);
```

```typescript
// frontend/components/auth/login-button.tsx
'use client';

import { signIn, signOut, useSession } from 'next-auth/react';
import { Button } from '@/components/ui/button';

export function LoginButton() {
  const { data: session } = useSession();

  if (session) {
    return (
      <Button onClick={() => signOut()}>
        Sign out
      </Button>
    );
  }

  return (
    <Button onClick={() => signIn('google')}>
      Sign in with Google
    </Button>
  );
}
```

### Phase 2: Report Configuration & Management

#### 2.1 Report Configuration Interface

```typescript
// frontend/types/report.ts
export interface Report {
  id: string;
  userId: string;
  title: string;
  description?: string;
  topics: string[];
  keywords: string[];
  frequency: 'daily' | 'weekly' | 'monthly';
  deliveryTime: string; // HH:mm format
  isActive: boolean;
  createdAt: string;
  updatedAt: string;
}

export interface ReportExecution {
  id: string;
  reportId: string;
  status: 'pending' | 'processing' | 'completed' | 'failed';
  startTime: string;
  endTime?: string;
  content?: string;
  error?: string;
}
```

```tsx
// frontend/components/reports/report-form.tsx
'use client';

import { useState } from 'react';
import { useForm } from 'react-hook-form';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Textarea } from '@/components/ui/textarea';
import { Select } from '@/components/ui/select';
import { Badge } from '@/components/ui/badge';
import type { Report } from '@/types/report';

interface ReportFormProps {
  report?: Partial<Report>;
  onSubmit: (data: Partial<Report>) => void;
}

export function ReportForm({ report, onSubmit }: ReportFormProps) {
  const [topics, setTopics] = useState<string[]>(report?.topics || []);
  const [currentTopic, setCurrentTopic] = useState('');

  const { register, handleSubmit, formState: { errors } } = useForm({
    defaultValues: report,
  });

  const addTopic = () => {
    if (currentTopic.trim() && !topics.includes(currentTopic.trim())) {
      setTopics([...topics, currentTopic.trim()]);
      setCurrentTopic('');
    }
  };

  const removeTopic = (topic: string) => {
    setTopics(topics.filter(t => t !== topic));
  };

  const onFormSubmit = (data: any) => {
    onSubmit({ ...data, topics });
  };

  return (
    <form onSubmit={handleSubmit(onFormSubmit)} className="space-y-6">
      <div>
        <label className="block text-sm font-medium mb-2">Report Title</label>
        <Input
          {...register('title', { required: 'Title is required' })}
          placeholder="e.g., Apple Inc. Daily News"
        />
        {errors.title && (
          <p className="text-red-500 text-sm mt-1">{errors.title.message}</p>
        )}
      </div>

      <div>
        <label className="block text-sm font-medium mb-2">Description</label>
        <Textarea
          {...register('description')}
          placeholder="Brief description of what this report should cover"
        />
      </div>

      <div>
        <label className="block text-sm font-medium mb-2">Topics</label>
        <div className="flex gap-2 mb-2">
          <Input
            value={currentTopic}
            onChange={(e) => setCurrentTopic(e.target.value)}
            placeholder="Add a topic (company, product, person)"
            onKeyPress={(e) => e.key === 'Enter' && (e.preventDefault(), addTopic())}
          />
          <Button type="button" onClick={addTopic}>Add</Button>
        </div>
        <div className="flex flex-wrap gap-2">
          {topics.map((topic) => (
            <Badge key={topic} variant="secondary" className="cursor-pointer"
              onClick={() => removeTopic(topic)}>
              {topic} ×
            </Badge>
          ))}
        </div>
      </div>

      <div>
        <label className="block text-sm font-medium mb-2">Frequency</label>
        <Select {...register('frequency', { required: true })}>
          <option value="daily">Daily</option>
          <option value="weekly">Weekly</option>
          <option value="monthly">Monthly</option>
        </Select>
      </div>

      <Button type="submit" className="w-full">
        {report ? 'Update Report' : 'Create Report'}
      </Button>
    </form>
  );
}
```

#### 2.2 Backend Report Management

```typescript
// backend/functions/reports/handler.ts
import { APIGatewayProxyHandler } from 'aws-lambda';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, PutCommand, QueryCommand, UpdateCommand } from '@aws-sdk/lib-dynamodb';
import { v4 as uuidv4 } from 'uuid';

const client = new DynamoDBClient({});
const ddbDocClient = DynamoDBDocumentClient.from(client);

export const createReport: APIGatewayProxyHandler = async (event) => {
  try {
    const { userId } = JSON.parse(event.requestContext.authorizer?.claims || '{}');
    const reportData = JSON.parse(event.body || '{}');
    
    const report = {
      id: uuidv4(),
      userId,
      ...reportData,
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
      isActive: true,
    };

    await ddbDocClient.send(new PutCommand({
      TableName: process.env.TABLE_NAME,
      Item: {
        PK: `USER#${userId}`,
        SK: `REPORT#${report.id}`,
        ...report,
      },
    }));

    return {
      statusCode: 200,
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(report),
    };
  } catch (error) {
    console.error('Error creating report:', error);
    return {
      statusCode: 500,
      body: JSON.stringify({ error: 'Failed to create report' }),
    };
  }
};

export const getUserReports: APIGatewayProxyHandler = async (event) => {
  try {
    const { userId } = JSON.parse(event.requestContext.authorizer?.claims || '{}');
    
    const result = await ddbDocClient.send(new QueryCommand({
      TableName: process.env.TABLE_NAME,
      KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
      ExpressionAttributeValues: {
        ':pk': `USER#${userId}`,
        ':sk': 'REPORT#',
      },
    }));

    const reports = result.Items || [];

    return {
      statusCode: 200,
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(reports),
    };
  } catch (error) {
    console.error('Error fetching reports:', error);
    return {
      statusCode: 500,
      body: JSON.stringify({ error: 'Failed to fetch reports' }),
    };
  }
};
```

### Phase 3: AI-Powered News Research

#### 3.1 Bedrock Integration

```typescript
// backend/functions/ai/news-research.ts
import { BedrockRuntimeClient, InvokeModelCommand } from '@aws-sdk/client-bedrock-runtime';

const bedrockClient = new BedrockRuntimeClient({ region: 'us-east-1' });

interface NewsResearchParams {
  topics: string[];
  keywords?: string[];
  dateRange?: string;
}

export async function researchNews(params: NewsResearchParams): Promise<string> {
  const prompt = `
You are a professional news researcher and writer. Your task is to research and compile a comprehensive news report.

Topics to research: ${params.topics.join(', ')}
${params.keywords ? `Additional keywords: ${params.keywords.join(', ')}` : ''}
Date range: ${params.dateRange || 'Last 24 hours'}

Instructions:
1. Search for recent news articles related to the specified topics
2. Analyze and summarize the most important developments
3. Organize information by topic/company
4. Include source attribution where possible
5. Highlight significant trends or changes
6. Format as a professional news brief

Format the response as a structured report with:
- Executive Summary
- Topic-specific sections
- Key Developments
- Market Impact (if applicable)
- Sources and References

Please conduct thorough research and provide accurate, up-to-date information.
`;

  try {
    const response = await bedrockClient.send(new InvokeModelCommand({
      modelId: 'anthropic.claude-3-5-sonnet-20241022-v2:0',
      body: JSON.stringify({
        anthropic_version: 'bedrock-2023-05-31',
        max_tokens: 4000,
        messages: [
          {
            role: 'user',
            content: prompt,
          },
        ],
      }),
      contentType: 'application/json',
      accept: 'application/json',
    }));

    const responseBody = JSON.parse(new TextDecoder().decode(response.body));
    return responseBody.content[0].text;
  } catch (error) {
    console.error('Error with Bedrock:', error);
    throw new Error('Failed to generate news report');
  }
}

// Alternative OpenAI implementation
export async function researchNewsOpenAI(params: NewsResearchParams): Promise<string> {
  const openai = new OpenAI({
    apiKey: process.env.OPENAI_API_KEY,
  });

  const completion = await openai.chat.completions.create({
    model: 'gpt-4',
    messages: [
      {
        role: 'system',
        content: 'You are a professional news researcher with access to current web search capabilities. Research and compile comprehensive news reports.',
      },
      {
        role: 'user',
        content: `Research recent news for: ${params.topics.join(', ')}. ${params.keywords ? `Focus on: ${params.keywords.join(', ')}` : ''}`,
      },
    ],
    max_tokens: 4000,
    tools: [
      {
        type: 'function',
        function: {
          name: 'search_web',
          description: 'Search the web for recent news articles',
          parameters: {
            type: 'object',
            properties: {
              query: { type: 'string' },
              time_range: { type: 'string' },
            },
          },
        },
      },
    ],
  });

  return completion.choices[0].message.content || '';
}
```

#### 3.2 Report Generation Lambda

```typescript
// backend/functions/reports/generator.ts
import { EventBridgeHandler } from 'aws-lambda';
import { DynamoDBDocumentClient, QueryCommand, PutCommand, UpdateCommand } from '@aws-sdk/lib-dynamodb';
import { SESv2Client, SendEmailCommand } from '@aws-sdk/client-sesv2';
import { researchNews } from '../ai/news-research';

const ddbDocClient = DynamoDBDocumentClient.from(new DynamoDBClient({}));
const sesClient = new SESv2Client({});

export const generateScheduledReport: EventBridgeHandler<any, any, void> = async (event) => {
  try {
    const { reportId } = event.detail;
    
    // Get report configuration
    const reportResult = await ddbDocClient.send(new QueryCommand({
      TableName: process.env.TABLE_NAME,
      KeyConditionExpression: 'PK = :pk AND SK = :sk',
      ExpressionAttributeValues: {
        ':pk': `USER#${event.detail.userId}`,
        ':sk': `REPORT#${reportId}`,
      },
    }));

    const report = reportResult.Items?.[0];
    if (!report || !report.isActive) {
      console.log('Report not found or inactive');
      return;
    }

    // Create execution record
    const executionId = uuidv4();
    const execution = {
      id: executionId,
      reportId,
      status: 'processing',
      startTime: new Date().toISOString(),
    };

    await ddbDocClient.send(new PutCommand({
      TableName: process.env.TABLE_NAME,
      Item: {
        PK: `REPORT#${reportId}`,
        SK: `EXECUTION#${new Date().toISOString().split('T')[0]}`,
        ...execution,
      },
    }));

    try {
      // Generate report content
      const content = await researchNews({
        topics: report.topics,
        keywords: report.keywords,
        dateRange: 'Last 24 hours',
      });

      // Update execution with success
      await ddbDocClient.send(new UpdateCommand({
        TableName: process.env.TABLE_NAME,
        Key: {
          PK: `REPORT#${reportId}`,
          SK: `EXECUTION#${new Date().toISOString().split('T')[0]}`,
        },
        UpdateExpression: 'SET #status = :status, endTime = :endTime, content = :content',
        ExpressionAttributeNames: {
          '#status': 'status',
        },
        ExpressionAttributeValues: {
          ':status': 'completed',
          ':endTime': new Date().toISOString(),
          ':content': content,
        },
      }));

      // Send email
      await sendReportEmail(report, content);

    } catch (error) {
      // Update execution with failure
      await ddbDocClient.send(new UpdateCommand({
        TableName: process.env.TABLE_NAME,
        Key: {
          PK: `REPORT#${reportId}`,
          SK: `EXECUTION#${new Date().toISOString().split('T')[0]}`,
        },
        UpdateExpression: 'SET #status = :status, endTime = :endTime, error = :error',
        ExpressionAttributeNames: {
          '#status': 'status',
        },
        ExpressionAttributeValues: {
          ':status': 'failed',
          ':endTime': new Date().toISOString(),
          ':error': error.message,
        },
      }));
    }
  } catch (error) {
    console.error('Error generating report:', error);
  }
};

async function sendReportEmail(report: any, content: string) {
  const htmlContent = `
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="utf-8">
      <title>${report.title}</title>
      <style>
        body { font-family: Arial, sans-serif; line-height: 1.6; color: #333; }
        .header { background: #f8f9fa; padding: 20px; border-radius: 8px; margin-bottom: 20px; }
        .content { white-space: pre-wrap; }
        .footer { margin-top: 30px; padding-top: 20px; border-top: 1px solid #eee; font-size: 14px; color: #666; }
      </style>
    </head>
    <body>
      <div class="header">
        <h1>${report.title}</h1>
        <p>Generated on ${new Date().toLocaleDateString()}</p>
      </div>
      <div class="content">${content}</div>
      <div class="footer">
        <p>This report was generated by NewsFlow. To manage your reports, visit your dashboard.</p>
      </div>
    </body>
    </html>
  `;

  await sesClient.send(new SendEmailCommand({
    FromEmailAddress: process.env.SES_FROM_EMAIL,
    Destination: {
      ToAddresses: [report.userEmail],
    },
    Content: {
      Simple: {
        Subject: {
          Data: `${report.title} - ${new Date().toLocaleDateString()}`,
        },
        Body: {
          Html: {
            Data: htmlContent,
          },
          Text: {
            Data: content,
          },
        },
      },
    },
  }));
}
```

### Phase 4: Scheduling & Delivery

#### 4.1 EventBridge Scheduler Setup

```typescript
// backend/functions/scheduler/setup.ts
import { EventBridgeClient, PutRuleCommand, PutTargetsCommand } from '@aws-sdk/client-eventbridge';

const eventBridge = new EventBridgeClient({});

export async function scheduleReport(report: any) {
  const ruleName = `newsflow-report-${report.id}`;
  
  // Convert frequency to cron expression
  const cronExpression = getCronExpression(report.frequency, report.deliveryTime);
  
  // Create the rule
  await eventBridge.send(new PutRuleCommand({
    Name: ruleName,
    ScheduleExpression: cronExpression,
    Description: `Schedule for report: ${report.title}`,
    State: report.isActive ? 'ENABLED' : 'DISABLED',
  }));

  // Add target (Lambda function)
  await eventBridge.send(new PutTargetsCommand({
    Rule: ruleName,
    Targets: [
      {
        Id: '1',
        Arn: process.env.REPORT_GENERATOR_LAMBDA_ARN,
        Input: JSON.stringify({
          reportId: report.id,
          userId: report.userId,
        }),
      },
    ],
  }));
}

function getCronExpression(frequency: string, deliveryTime: string): string {
  const [hour, minute] = deliveryTime.split(':');
  
  switch (frequency) {
    case 'daily':
      return `cron(${minute} ${hour} * * ? *)`;
    case 'weekly':
      return `cron(${minute} ${hour} ? * MON *)`;
    case 'monthly':
      return `cron(${minute} ${hour} 1 * ? *)`;
    default:
      throw new Error('Invalid frequency');
  }
}
```

## 🎨 Frontend Implementation Details

### Dashboard Components

```tsx
// frontend/components/dashboard/report-list.tsx
'use client';

import { useState, useEffect } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Switch } from '@/components/ui/switch';
import type { Report } from '@/types/report';

export function ReportList() {
  const [reports, setReports] = useState<Report[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchReports();
  }, []);

  const fetchReports = async () => {
    try {
      const response = await fetch('/api/reports');
      const data = await response.json();
      setReports(data);
    } catch (error) {
      console.error('Error fetching reports:', error);
    } finally {
      setLoading(false);
    }
  };

  const toggleReport = async (reportId: string, isActive: boolean) => {
    try {
      await fetch(`/api/reports/${reportId}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ isActive }),
      });
      fetchReports(); // Refresh list
    } catch (error) {
      console.error('Error updating report:', error);
    }
  };

  if (loading) return <div>Loading reports...</div>;

  return (
    <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
      {reports.map((report) => (
        <Card key={report.id}>
          <CardHeader>
            <div className="flex items-center justify-between">
              <CardTitle className="text-lg">{report.title}</CardTitle>
              <Switch
                checked={report.isActive}
                onCheckedChange={(checked) => toggleReport(report.id, checked)}
              />
            </div>
          </CardHeader>
          <CardContent>
            <div className="space-y-2">
              <div className="flex flex-wrap gap-1">
                {report.topics.map((topic) => (
                  <Badge key={topic} variant="outline" className="text-xs">
                    {topic}
                  </Badge>
                ))}
              </div>
              <p className="text-sm text-gray-600">
                Frequency: {report.frequency}
              </p>
              <p className="text-sm text-gray-600">
                Delivery: {report.deliveryTime}
              </p>
              <div className="flex gap-2 mt-4">
                <Button size="sm" variant="outline">
                  Edit
                </Button>
                <Button size="sm" variant="outline">
                  View Reports
                </Button>
              </div>
            </div>
          </CardContent>
        </Card>
      ))}
    </div>
  );
}
```

## 🧪 Testing Strategy

### Unit Tests

```typescript
// tests/ai/news-research.test.ts
import { describe, it, expect, vi } from 'vitest';
import { researchNews } from '../../backend/functions/ai/news-research';

vi.mock('@aws-sdk/client-bedrock-runtime');

describe('News Research', () => {
  it('should generate news report for given topics', async () => {
    const params = {
      topics: ['Apple Inc.', 'iPhone'],
      keywords: ['earnings', 'stock'],
    };

    const result = await researchNews(params);
    
    expect(result).toBeDefined();
    expect(typeof result).toBe('string');
    expect(result.length).toBeGreaterThan(100);
  });

  it('should handle errors gracefully', async () => {
    const params = {
      topics: [],
    };

    await expect(researchNews(params)).rejects.toThrow();
  });
});
```

### Integration Tests

```typescript
// tests/integration/report-lifecycle.test.ts
import { describe, it, expect } from 'vitest';
import { createReport, generateReport } from '../test-utils';

describe('Report Lifecycle', () => {
  it('should create, schedule, and generate report', async () => {
    // Create report
    const report = await createReport({
      title: 'Test Report',
      topics: ['OpenAI'],
      frequency: 'daily',
      deliveryTime: '09:00',
    });

    expect(report.id).toBeDefined();
    expect(report.isActive).toBe(true);

    // Generate report
    const execution = await generateReport(report.id);
    
    expect(execution.status).toBe('completed');
    expect(execution.content).toBeDefined();
  });
});
```

## 🚀 Deployment Instructions

### AWS CDK Deployment

```bash
# Install dependencies
npm install

# Bootstrap CDK (first time only)
npx cdk bootstrap

# Deploy infrastructure
npx cdk deploy NewsFlowStack

# Deploy Lambda functions
npm run deploy:functions

# Deploy frontend
npm run deploy:frontend
```

### Environment Configuration

```bash
# Set up environment variables
aws ssm put-parameter --name "/newsflow/google-client-id" --value "your-google-client-id" --type "SecureString"
aws ssm put-parameter --name "/newsflow/google-client-secret" --value "your-google-client-secret" --type "SecureString"
aws ssm put-parameter --name "/newsflow/openai-api-key" --value "your-openai-api-key" --type "SecureString"

# Configure SES
aws ses verify-email-identity --email-address noreply@yourdomain.com
```

## 🔧 Troubleshooting Guide

### Common Issues

1. **Bedrock Access Denied**
   - Ensure your AWS account has access to Bedrock
   - Check IAM permissions for the Lambda execution role

2. **DynamoDB Throttling**
   - Enable auto-scaling or increase provisioned capacity
   - Implement exponential backoff in your code

3. **Email Delivery Issues**
   - Verify SES email addresses and domains
   - Check SES sending limits and reputation

4. **Authentication Problems**
   - Verify Google OAuth configuration
   - Check CORS settings in API Gateway

### Monitoring & Logging

```typescript
// backend/shared/logger.ts
import { Logger } from '@aws-lambda-powertools/logger';

export const logger = new Logger({
  serviceName: 'newsflow',
  logLevel: process.env.LOG_LEVEL || 'INFO',
});

export function logApiCall(event: any, context: any) {
  logger.info('API call', {
    requestId: context.awsRequestId,
    path: event.path,
    method: event.httpMethod,
    userAgent: event.headers?.['user-agent'],
  });
}
```

## 📈 Performance Optimization

### Lambda Cold Start Reduction

```typescript
// backend/functions/shared/init.ts
// Initialize connections outside handler for reuse
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { BedrockRuntimeClient } from '@aws-sdk/client-bedrock-runtime';

export const ddbClient = new DynamoDBClient({});
export const bedrockClient = new BedrockRuntimeClient({ region: 'us-east-1' });

// Use provisioned concurrency for critical functions
export const handler = async (event: any) => {
  // Handler implementation
};
```

### Caching Strategy

```typescript
// backend/shared/cache.ts
import { ElastiCacheClient } from '@aws-sdk/client-elasticache';

class CacheService {
  private client = new ElastiCacheClient({});
  
  async get(key: string): Promise<any> {
    // Implementation
  }
  
  async set(key: string, value: any, ttl: number): Promise<void> {
    // Implementation
  }
}

export const cache = new CacheService();
```

## 🔒 Security Best Practices

### Input Validation

```typescript
// backend/shared/validation.ts
import Joi from 'joi';

export const reportSchema = Joi.object({
  title: Joi.string().min(1).max(100).required(),
  topics: Joi.array().items(Joi.string().min(1).max(50)).min(1).max(10).required(),
  frequency: Joi.string().valid('daily', 'weekly', 'monthly').required(),
  deliveryTime: Joi.string().pattern(/^([01]?[0-9]|2[0-3]):[0-5][0-9]$/).required(),
});

export function validateReport(data: any) {
  const { error, value } = reportSchema.validate(data);
  if (error) {
    throw new Error(`Validation error: ${error.message}`);
  }
  return value;
}
```

### Rate Limiting

```typescript
// backend/shared/rate-limiter.ts
export async function checkRateLimit(userId: string, action: string): Promise<boolean> {
  const key = `rate_limit:${userId}:${action}`;
  const current = await cache.get(key) || 0;
  
  const limits = {
    create_report: 10, // per hour
    generate_report: 50, // per hour
  };
  
  if (current >= limits[action]) {
    return false;
  }
  
  await cache.set(key, current + 1, 3600); // 1 hour TTL
  return true;
}
```

This comprehensive guide provides everything needed to build the NewsFlow application. Use this document as a reference throughout development, and feel free to extend or modify based on specific requirements.

Remember to:
1. Test thoroughly at each phase
2. Monitor performance and costs
3. Implement proper error handling
4. Follow security best practices
5. Document any changes or extensions