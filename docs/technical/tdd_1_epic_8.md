# Technical Design Document – Epic 8: AI & Analytics Features

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 8 – AI & Analytics Features
- **Source**: Derived from `fdd_1_agile.md` Epic 8 (Stories CRM-033 through CRM-035)
- **Reference Documents**: 
  - `fdd_1.md` (Functional Design Document for CRM, §5)
  - `fdd_1_agile.md` (Agile User Stories)
  - `tdd_1_epic_1.md` (CRM Core Data Model - prerequisite)
  - `tdd_1_epic_2.md` (Authentication, Authorization & RLS Policies - prerequisite)
  - `tdd_1_epic_3.md` (Customer Management APIs - prerequisite)
  - `tdd_1_epic_4.md` (Interaction & Communication Logging - prerequisite)
  - `tdd_1_epic_5.md` (Follow-Ups & Reminders - prerequisite)
  - `tdd_1_epic_6.md` (Segmentation & Targeting - prerequisite)
  - `tdd_1_epic_7.md` (Automation Engine - prerequisite)
  - `tech_features.md` (Advanced AI Features)
  - `functional.md` (Platform Capabilities)
  - `tooling.md` (Platform Choices: Supabase + Vercel)
- **Target Platform**: Supabase (PostgreSQL 15+ with Edge Functions)
- **Purpose**: Comprehensive technical specification for LLM-based code generation
- **Prerequisites**: Epic 1-7 must be completed first

---

## 1. Overview

This document provides complete technical specifications for implementing AI-driven features in Supabase. It covers:

- AI-Driven Customer Segmentation Function for generating segments from natural language prompts
- Sentiment Analysis for Interactions to classify customer communication sentiment
- AI-Based Follow-Up Suggestions for proactive customer engagement
- AI provider integration patterns (OpenAI, Anthropic, or other LLM providers)
- Privacy and security considerations for PII handling
- Error handling and graceful degradation
- Cost management and rate limiting
- Request/response schemas with exact JSON structures
- Database triggers and background processing mechanisms

All specifications are designed to be directly implementable as Supabase Edge Functions (Deno/TypeScript), with exact schemas, validation rules, AI provider integration patterns, and error codes defined.

---

## 2. Architecture Decisions

### 2.1 AI Provider Selection

**Decision**: Support multiple AI providers with provider abstraction layer:

- **Primary Provider**: OpenAI GPT-4 or GPT-3.5-turbo (recommended for production)
- **Alternative Providers**: Anthropic Claude, Google Gemini (via abstraction layer)
- **Fallback Strategy**: Graceful degradation when AI provider is unavailable

**Rationale**: 
- Provider abstraction allows switching providers without code changes
- Multiple providers provide redundancy and cost optimization
- Different providers may excel at different tasks

### 2.2 Privacy & Security

**Decision**: Minimize PII exposure to AI providers:

- **Aggregated Data**: Send aggregated metrics instead of raw customer data when possible
- **Anonymization**: Remove or hash PII before sending to AI
- **Data Minimization**: Only send necessary fields for AI processing
- **Documentation**: Clear documentation of what data is sent to AI providers

**Implementation**:
- Create aggregation functions that summarize customer data without exposing PII
- Use field-level filtering to exclude sensitive data
- Log all AI API calls for audit purposes

### 2.3 Error Handling & Fallbacks

**Decision**: Implement graceful degradation:

- **AI Failures**: Do not block core functionality
- **Timeouts**: Set reasonable timeouts (30 seconds for AI calls)
- **Rate Limits**: Implement exponential backoff and retry logic
- **Fallback Values**: Use safe defaults when AI fails (e.g., `sentiment = null`)

### 2.4 Cost Management

**Decision**: Implement cost controls:

- **Rate Limiting**: Limit AI calls per org per day/hour
- **Caching**: Cache AI results when appropriate
- **Batching**: Batch requests when possible
- **Cost Tracking**: Log token usage and costs per org

### 2.5 Processing Patterns

**Decision**: Use async processing for AI operations:

- **Synchronous**: Simple operations (sentiment analysis) can be synchronous
- **Asynchronous**: Complex operations (segmentation, follow-up suggestions) use async queues
- **Database Triggers**: Use PostgreSQL triggers to enqueue AI processing
- **Background Jobs**: Use Supabase cron or Edge Function queues for batch processing

---

## 3. Story CRM-033: AI-Driven Customer Segmentation Function

### 3.1 Function Specification

**Path**: `POST /crm/segments/:id/compute-ai` (called internally or via recompute endpoint)

**Method**: `POST`

**Authentication**: Service Role Key (internal) or Authenticated User (manual trigger)

**Purpose**: Generate customer segment membership using AI based on natural language prompt

### 3.2 Request Schema

```typescript
interface ComputeAISegmentRequest {
  segment_id: string; // UUID
  // Optional: override ai_prompt
  ai_prompt_override?: string;
  // Optional: limit number of customers to analyze (for cost control)
  max_customers?: number; // Default: 1000, max: 10000
}
```

### 3.3 AI Provider Integration

#### 3.3.1 Provider Abstraction Layer

**File**: `supabase/functions/_shared/ai-provider.ts`

```typescript
export interface AIProvider {
  generateSegment(prompt: string, schema: string, sampleData: any[]): Promise<AISegmentResult>;
  analyzeSentiment(text: string): Promise<SentimentResult>;
  suggestFollowups(customerHistory: CustomerHistory): Promise<FollowUpSuggestion[]>;
}

export interface AISegmentResult {
  definition?: {
    operator: 'AND' | 'OR';
    rules: Array<{
      field: string;
      operator: string;
      value: any;
    }>;
  };
  scores?: Array<{
    customer_id: string;
    score: number;
    explanation: string;
  }>;
  explanation: string; // Human-readable explanation
  method: 'rule_based' | 'scoring'; // How AI generated the segment
}

export interface SentimentResult {
  sentiment: 'positive' | 'neutral' | 'negative';
  confidence: number; // 0.0 to 1.0
  explanation?: string;
}

export interface FollowUpSuggestion {
  title: string;
  description: string;
  due_at_offset_minutes: number;
  priority: 'low' | 'medium' | 'high';
  reasoning: string;
}

// OpenAI Implementation
export class OpenAIProvider implements AIProvider {
  private apiKey: string;
  private baseUrl: string = 'https://api.openai.com/v1';
  
  constructor(apiKey: string) {
    this.apiKey = apiKey;
  }
  
  async generateSegment(
    prompt: string,
    schema: string,
    sampleData: any[]
  ): Promise<AISegmentResult> {
    const systemPrompt = `You are a CRM segmentation assistant. Your task is to analyze customer data and create segments based on natural language prompts.

Database Schema:
${schema}

Sample Customer Data (anonymized):
${JSON.stringify(sampleData.slice(0, 10), null, 2)}

Instructions:
1. Analyze the prompt and determine what customer segment is being requested
2. Generate either:
   a) A rule-based segment definition (JSON format matching the schema)
   b) A scoring-based segment with per-customer scores
3. Provide a human-readable explanation of the segment
4. If generating rules, ensure they are valid and can be executed against the database schema

Output Format:
{
  "method": "rule_based" | "scoring",
  "definition": { ... } (if rule_based),
  "scores": [ ... ] (if scoring),
  "explanation": "Human-readable explanation"
}`;

    const response = await fetch(`${this.baseUrl}/chat/completions`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        model: 'gpt-4-turbo-preview', // or gpt-3.5-turbo for cost savings
        messages: [
          { role: 'system', content: systemPrompt },
          { role: 'user', content: prompt }
        ],
        temperature: 0.3, // Lower temperature for more consistent results
        max_tokens: 2000,
      }),
    });
    
    if (!response.ok) {
      const error = await response.json();
      throw new Error(`OpenAI API error: ${error.error?.message || 'Unknown error'}`);
    }
    
    const data = await response.json();
    const content = data.choices[0].message.content;
    
    // Parse JSON response
    try {
      const result = JSON.parse(content);
      return result;
    } catch (e) {
      // Fallback: try to extract JSON from markdown code blocks
      const jsonMatch = content.match(/```json\n([\s\S]*?)\n```/);
      if (jsonMatch) {
        return JSON.parse(jsonMatch[1]);
      }
      throw new Error('Failed to parse AI response as JSON');
    }
  }
  
  async analyzeSentiment(text: string): Promise<SentimentResult> {
    const response = await fetch(`${this.baseUrl}/chat/completions`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        model: 'gpt-3.5-turbo', // Use cheaper model for sentiment
        messages: [
          {
            role: 'system',
            content: 'You are a sentiment analysis assistant. Analyze the sentiment of customer communications and respond with JSON: {"sentiment": "positive" | "neutral" | "negative", "confidence": 0.0-1.0, "explanation": "brief explanation"}.'
          },
          {
            role: 'user',
            content: `Analyze the sentiment of this customer communication:\n\n${text}`
          }
        ],
        temperature: 0.1,
        max_tokens: 150,
      }),
    });
    
    if (!response.ok) {
      throw new Error('OpenAI API error');
    }
    
    const data = await response.json();
    const content = data.choices[0].message.content;
    
    try {
      return JSON.parse(content);
    } catch (e) {
      // Fallback parsing
      const jsonMatch = content.match(/```json\n([\s\S]*?)\n```/);
      if (jsonMatch) {
        return JSON.parse(jsonMatch[1]);
      }
      // Last resort: simple keyword-based fallback
      return {
        sentiment: inferSentimentFromKeywords(text),
        confidence: 0.5,
        explanation: 'Fallback sentiment analysis'
      };
    }
  }
  
  async suggestFollowups(customerHistory: CustomerHistory): Promise<FollowUpSuggestion[]> {
    const systemPrompt = `You are a CRM assistant that suggests follow-up actions for customers based on their history.

Analyze the customer's history and suggest 1-3 follow-up actions with:
- Title: Brief action title
- Description: What should be done
- due_at_offset_minutes: Minutes from now (suggest realistic timing)
- priority: low, medium, or high
- reasoning: Why this follow-up is recommended

Return JSON array of suggestions.`;

    const response = await fetch(`${this.baseUrl}/chat/completions`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        model: 'gpt-4-turbo-preview',
        messages: [
          { role: 'system', content: systemPrompt },
          { role: 'user', content: JSON.stringify(customerHistory, null, 2) }
        ],
        temperature: 0.4,
        max_tokens: 1000,
      }),
    });
    
    if (!response.ok) {
      throw new Error('OpenAI API error');
    }
    
    const data = await response.json();
    const content = data.choices[0].message.content;
    
    try {
      return JSON.parse(content);
    } catch (e) {
      const jsonMatch = content.match(/```json\n([\s\S]*?)\n```/);
      if (jsonMatch) {
        return JSON.parse(jsonMatch[1]);
      }
      return []; // Return empty array on parse failure
    }
  }
}

function inferSentimentFromKeywords(text: string): 'positive' | 'neutral' | 'negative' {
  const lowerText = text.toLowerCase();
  const positiveWords = ['thank', 'great', 'excellent', 'happy', 'satisfied', 'love', 'appreciate'];
  const negativeWords = ['angry', 'frustrated', 'disappointed', 'terrible', 'awful', 'horrible', 'complaint'];
  
  const positiveCount = positiveWords.filter(w => lowerText.includes(w)).length;
  const negativeCount = negativeWords.filter(w => lowerText.includes(w)).length;
  
  if (negativeCount > positiveCount) return 'negative';
  if (positiveCount > negativeCount) return 'positive';
  return 'neutral';
}
```

### 3.4 Implementation: Edge Function

**File**: `supabase/functions/crm-ai-compute-segment/index.ts`

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';
import { OpenAIProvider, AIProvider } from '../_shared/ai-provider.ts';

serve(async (req) => {
  try {
    const supabaseAdmin = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? '',
    );
    
    const payload = await req.json();
    const segmentId = payload.segment_id;
    
    if (!segmentId) {
      return new Response(
        JSON.stringify({ error: 'segment_id is required' }),
        { status: 400, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    // Get segment
    const { data: segment, error: segmentError } = await supabaseAdmin
      .from('crm_segments')
      .select('*')
      .eq('id', segmentId)
      .single();
    
    if (segmentError || !segment) {
      return new Response(
        JSON.stringify({ error: 'Segment not found' }),
        { status: 404, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    if (segment.type !== 'ai_generated') {
      return new Response(
        JSON.stringify({ error: 'Segment is not ai_generated type' }),
        { status: 400, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    const aiPrompt = payload.ai_prompt_override || segment.ai_prompt;
    if (!aiPrompt) {
      return new Response(
        JSON.stringify({ error: 'ai_prompt is required' }),
        { status: 400, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    // Check rate limits (implement rate limiting logic)
    const rateLimitCheck = await checkAIRateLimit(supabaseAdmin, segment.org_id);
    if (!rateLimitCheck.allowed) {
      return new Response(
        JSON.stringify({ error: 'AI rate limit exceeded', retry_after: rateLimitCheck.retry_after }),
        { status: 429, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    // Get aggregated customer data (anonymized)
    const maxCustomers = Math.min(payload.max_customers || 1000, 10000);
    const { data: customers, error: customersError } = await supabaseAdmin.rpc(
      'crm_get_aggregated_customer_data',
      {
        p_org_id: segment.org_id,
        p_limit: maxCustomers,
      }
    );
    
    if (customersError) {
      return new Response(
        JSON.stringify({ error: customersError.message }),
        { status: 500, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    // Build schema description
    const schemaDescription = buildSchemaDescription();
    
    // Initialize AI provider
    const aiProvider = new OpenAIProvider(Deno.env.get('OPENAI_API_KEY') || '');
    
    // Call AI
    let aiResult;
    try {
      aiResult = await aiProvider.generateSegment(aiPrompt, schemaDescription, customers);
    } catch (error) {
      // Update segment with error
      await supabaseAdmin
        .from('crm_segments')
        .update({
          ai_explanation: `Error: ${error.message}`,
          last_computed_at: new Date().toISOString(),
        })
        .eq('id', segmentId);
      
      return new Response(
        JSON.stringify({ error: `AI processing failed: ${error.message}` }),
        { status: 500, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    // Process AI result
    if (aiResult.method === 'rule_based' && aiResult.definition) {
      // Update segment with rule definition
      await supabaseAdmin
        .from('crm_segments')
        .update({
          definition: aiResult.definition,
          ai_explanation: aiResult.explanation,
          last_computed_at: new Date().toISOString(),
        })
        .eq('id', segmentId);
      
      // Compute members using rule-based logic (from Epic 6)
      await supabaseAdmin.rpc('crm_compute_segment_members', {
        p_segment_id: segmentId,
        p_org_id: segment.org_id,
      });
    } else if (aiResult.method === 'scoring' && aiResult.scores) {
      // Store scores in segment_members
      await supabaseAdmin
        .from('crm_segments')
        .update({
          ai_explanation: aiResult.explanation,
          last_computed_at: new Date().toISOString(),
        })
        .eq('id', segmentId);
      
      // Insert/update segment members with scores
      for (const scoreEntry of aiResult.scores) {
        await supabaseAdmin
          .from('crm_segment_members')
          .upsert({
            org_id: segment.org_id,
            segment_id: segmentId,
            customer_id: scoreEntry.customer_id,
            score: scoreEntry.score,
            metadata: {
              explanation: scoreEntry.explanation,
              ai_generated: true,
            },
          }, {
            onConflict: 'org_id,segment_id,customer_id',
          });
      }
    }
    
    // Log AI usage
    await logAIUsage(supabaseAdmin, segment.org_id, 'segment_generation', {
      segment_id: segmentId,
      tokens_used: 0, // Would need to track from AI response
      cost_estimate: 0, // Would calculate based on tokens
    });
    
    return new Response(
      JSON.stringify({
        segment_id: segmentId,
        method: aiResult.method,
        member_count: aiResult.scores?.length || 0,
        explanation: aiResult.explanation,
      }),
      { status: 200, headers: { 'Content-Type': 'application/json' } }
    );
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { 'Content-Type': 'application/json' } }
    );
  }
});

function buildSchemaDescription(): string {
  return `
Customer Schema:
- id: UUID
- type: 'individual' | 'company'
- name: string
- status: 'active' | 'prospect' | 'inactive' | 'blacklisted'
- lifecycle_stage: 'lead' | 'opportunity' | 'customer' | 'former_customer'
- created_at: timestamp
- updated_at: timestamp

Related Data:
- tags: array of tag names
- locations: array of location cities/states
- interactions: count and last interaction date
- followups: count of pending follow-ups
`;
}

async function checkAIRateLimit(
  supabase: any,
  orgId: string
): Promise<{ allowed: boolean; retry_after?: number }> {
  // Check AI usage in last 24 hours
  const { data: usage } = await supabase
    .from('crm_ai_usage_log')
    .select('created_at')
    .eq('org_id', orgId)
    .gte('created_at', new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString());
  
  const dailyLimit = 100; // Configurable per org tier
  if (usage && usage.length >= dailyLimit) {
    return { allowed: false, retry_after: 3600 }; // Retry after 1 hour
  }
  
  return { allowed: true };
}

async function logAIUsage(
  supabase: any,
  orgId: string,
  operation: string,
  metadata: any
): Promise<void> {
  await supabase
    .from('crm_ai_usage_log')
    .insert({
      org_id: orgId,
      operation,
      metadata,
      created_at: new Date().toISOString(),
    });
}
```

### 3.5 Aggregated Customer Data Function

**File**: PostgreSQL RPC Function

```sql
CREATE OR REPLACE FUNCTION crm_get_aggregated_customer_data(
  p_org_id UUID,
  p_limit INTEGER DEFAULT 1000
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_result JSONB;
BEGIN
  -- Return aggregated, anonymized customer data for AI processing
  SELECT jsonb_agg(
    jsonb_build_object(
      'customer_id', c.id, -- Keep ID for matching, but don't expose other PII
      'type', c.type,
      'status', c.status,
      'lifecycle_stage', c.lifecycle_stage,
      'created_at', c.created_at,
      'tags', (
        SELECT jsonb_agg(t.name)
        FROM crm_customer_tags cct
        JOIN crm_tags t ON t.id = cct.tag_id
        WHERE cct.customer_id = c.id
      ),
      'interaction_count', (
        SELECT COUNT(*) FROM crm_interactions ci
        WHERE ci.customer_id = c.id
      ),
      'last_interaction_at', (
        SELECT MAX(occurred_at) FROM crm_interactions ci
        WHERE ci.customer_id = c.id
      ),
      'followup_count', (
        SELECT COUNT(*) FROM crm_followups cf
        WHERE cf.customer_id = c.id AND cf.status = 'pending'
      )
    )
  )
  INTO v_result
  FROM customers c
  WHERE c.org_id = p_org_id
  LIMIT p_limit;
  
  RETURN COALESCE(v_result, '[]'::jsonb);
END;
$$;
```

### 3.6 AI Usage Logging Table

**DDL**:

```sql
CREATE TABLE IF NOT EXISTS crm_ai_usage_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  operation TEXT NOT NULL, -- 'segment_generation', 'sentiment_analysis', 'followup_suggestion'
  metadata JSONB, -- Tokens used, cost estimate, etc.
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_crm_ai_usage_log_org_id_created_at ON crm_ai_usage_log(org_id, created_at DESC);
```

---

## 4. Story CRM-034: Sentiment Analysis for Interactions

### 4.1 Function Specification

**Path**: `POST /crm/interactions/:id/analyze-sentiment` (called automatically or manually)

**Method**: `POST`

**Authentication**: Service Role Key (internal) or Authenticated User (manual trigger)

**Purpose**: Analyze sentiment of interaction and update `crm_interactions.sentiment` field

### 4.2 Implementation: Edge Function

**File**: `supabase/functions/crm-ai-analyze-sentiment/index.ts`

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';
import { OpenAIProvider } from '../_shared/ai-provider.ts';

serve(async (req) => {
  try {
    const supabaseAdmin = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? '',
    );
    
    const url = new URL(req.url);
    const interactionId = url.pathname.split('/').pop();
    
    if (!interactionId) {
      return new Response(
        JSON.stringify({ error: 'Interaction ID is required' }),
        { status: 400, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    // Get interaction
    const { data: interaction, error: interactionError } = await supabaseAdmin
      .from('crm_interactions')
      .select('*')
      .eq('id', interactionId)
      .single();
    
    if (interactionError || !interaction) {
      return new Response(
        JSON.stringify({ error: 'Interaction not found' }),
        { status: 404, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    // Skip if sentiment already analyzed
    if (interaction.sentiment) {
      return new Response(
        JSON.stringify({ message: 'Sentiment already analyzed', sentiment: interaction.sentiment }),
        { status: 200, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    // Get text to analyze (prefer summary, fallback to body)
    const textToAnalyze = interaction.summary || interaction.body || interaction.subject || '';
    
    if (!textToAnalyze || textToAnalyze.trim().length === 0) {
      // No text to analyze, set to neutral
      await supabaseAdmin
        .from('crm_interactions')
        .update({ sentiment: 'neutral' })
        .eq('id', interactionId);
      
      return new Response(
        JSON.stringify({ sentiment: 'neutral', reason: 'no_text_to_analyze' }),
        { status: 200, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    // Check rate limits
    const rateLimitCheck = await checkAIRateLimit(supabaseAdmin, interaction.org_id);
    if (!rateLimitCheck.allowed) {
      // Don't fail, just skip analysis
      return new Response(
        JSON.stringify({ message: 'Rate limit exceeded, skipping analysis' }),
        { status: 200, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    // Initialize AI provider
    const aiProvider = new OpenAIProvider(Deno.env.get('OPENAI_API_KEY') || '');
    
    // Analyze sentiment with timeout
    let sentimentResult;
    try {
      const controller = new AbortController();
      const timeoutId = setTimeout(() => controller.abort(), 30000); // 30 second timeout
      
      sentimentResult = await aiProvider.analyzeSentiment(textToAnalyze);
      
      clearTimeout(timeoutId);
    } catch (error) {
      if (error.name === 'AbortError') {
        // Timeout - use fallback
        sentimentResult = {
          sentiment: inferSentimentFromKeywords(textToAnalyze),
          confidence: 0.5,
          explanation: 'Timeout fallback'
        };
      } else {
        // Other error - use fallback
        sentimentResult = {
          sentiment: inferSentimentFromKeywords(textToAnalyze),
          confidence: 0.5,
          explanation: `AI error fallback: ${error.message}`
        };
      }
    }
    
    // Update interaction
    await supabaseAdmin
      .from('crm_interactions')
      .update({
        sentiment: sentimentResult.sentiment,
        metadata: COALESCE(interaction.metadata, '{}'::jsonb) || jsonb_build_object(
          'sentiment_confidence', sentimentResult.confidence,
          'sentiment_explanation', sentimentResult.explanation,
          'sentiment_analyzed_at', now()
        )
      })
      .eq('id', interactionId);
    
    // Log AI usage
    await logAIUsage(supabaseAdmin, interaction.org_id, 'sentiment_analysis', {
      interaction_id: interactionId,
    });
    
    return new Response(
      JSON.stringify({
        interaction_id: interactionId,
        sentiment: sentimentResult.sentiment,
        confidence: sentimentResult.confidence,
        explanation: sentimentResult.explanation,
      }),
      { status: 200, headers: { 'Content-Type': 'application/json' } }
    );
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { 'Content-Type': 'application/json' } }
    );
  }
});

function inferSentimentFromKeywords(text: string): 'positive' | 'neutral' | 'negative' {
  const lowerText = text.toLowerCase();
  const positiveWords = ['thank', 'great', 'excellent', 'happy', 'satisfied', 'love', 'appreciate', 'wonderful', 'amazing'];
  const negativeWords = ['angry', 'frustrated', 'disappointed', 'terrible', 'awful', 'horrible', 'complaint', 'unhappy', 'poor'];
  
  const positiveCount = positiveWords.filter(w => lowerText.includes(w)).length;
  const negativeCount = negativeWords.filter(w => lowerText.includes(w)).length;
  
  if (negativeCount > positiveCount && negativeCount > 0) return 'negative';
  if (positiveCount > negativeCount && positiveCount > 0) return 'positive';
  return 'neutral';
}
```

### 4.3 Database Trigger for Automatic Sentiment Analysis

**DDL**:

```sql
-- Function to call Edge Function for sentiment analysis
CREATE OR REPLACE FUNCTION trigger_sentiment_analysis()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
  -- Only analyze if channel supports sentiment analysis and sentiment is not set
  IF NEW.sentiment IS NULL AND 
     NEW.channel IN ('email_inbound', 'email_outbound', 'sms_inbound', 'sms_outbound', 'portal_message') AND
     (NEW.body IS NOT NULL OR NEW.summary IS NOT NULL OR NEW.subject IS NOT NULL) THEN
    
    -- Call Edge Function asynchronously (non-blocking)
    -- Note: This requires pg_net extension or similar
    -- For now, use a queue table or scheduled job
    
    -- Alternative: Insert into queue table for background processing
    INSERT INTO crm_sentiment_analysis_queue (interaction_id, org_id, created_at)
    VALUES (NEW.id, NEW.org_id, now())
    ON CONFLICT (interaction_id) DO NOTHING;
  END IF;
  
  RETURN NEW;
END;
$$;

-- Create queue table
CREATE TABLE IF NOT EXISTS crm_sentiment_analysis_queue (
  interaction_id UUID PRIMARY KEY REFERENCES crm_interactions(id) ON DELETE CASCADE,
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  processed_at TIMESTAMPTZ,
  status TEXT DEFAULT 'pending' -- 'pending', 'processing', 'completed', 'failed'
);

CREATE INDEX idx_crm_sentiment_analysis_queue_org_id_status ON crm_sentiment_analysis_queue(org_id, status) 
  WHERE status = 'pending';

-- Trigger
CREATE TRIGGER trigger_crm_interactions_sentiment_analysis
  AFTER INSERT ON crm_interactions
  FOR EACH ROW
  EXECUTE FUNCTION trigger_sentiment_analysis();
```

### 4.4 Background Processor for Sentiment Queue

**File**: `supabase/functions/crm-ai-process-sentiment-queue/index.ts`

**Schedule**: Every 1 minute (cron job)

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

serve(async (req) => {
  const supabaseAdmin = createClient(
    Deno.env.get('SUPABASE_URL') ?? '',
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? '',
  );
  
  // Get pending items from queue
  const { data: queueItems } = await supabaseAdmin
    .from('crm_sentiment_analysis_queue')
    .select('interaction_id')
    .eq('status', 'pending')
    .limit(10) // Process 10 at a time
    .order('created_at', { ascending: true });
  
  if (!queueItems || queueItems.length === 0) {
    return new Response(JSON.stringify({ message: 'No items to process' }), {
      status: 200,
      headers: { 'Content-Type': 'application/json' }
    });
  }
  
  const results = [];
  
  for (const item of queueItems) {
    // Mark as processing
    await supabaseAdmin
      .from('crm_sentiment_analysis_queue')
      .update({ status: 'processing' })
      .eq('interaction_id', item.interaction_id);
    
    try {
      // Call sentiment analysis function
      const response = await fetch(
        `${Deno.env.get('SUPABASE_URL')}/functions/v1/crm-ai-analyze-sentiment`,
        {
          method: 'POST',
          headers: {
            'Authorization': `Bearer ${Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')}`,
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({ interaction_id: item.interaction_id }),
        }
      );
      
      if (response.ok) {
        await supabaseAdmin
          .from('crm_sentiment_analysis_queue')
          .update({ status: 'completed', processed_at: new Date().toISOString() })
          .eq('interaction_id', item.interaction_id);
        
        results.push({ interaction_id: item.interaction_id, status: 'success' });
      } else {
        throw new Error('Analysis failed');
      }
    } catch (error) {
      await supabaseAdmin
        .from('crm_sentiment_analysis_queue')
        .update({ status: 'failed', processed_at: new Date().toISOString() })
        .eq('interaction_id', item.interaction_id);
      
      results.push({ interaction_id: item.interaction_id, status: 'failed', error: error.message });
    }
  }
  
  return new Response(
    JSON.stringify({ processed: results.length, results }),
    { status: 200, headers: { 'Content-Type': 'application/json' } }
  );
});
```

---

## 5. Story CRM-035: AI-Based Follow-Up Suggestions

### 5.1 Function Specification

**Path**: `POST /crm/customers/:id/suggest-followups`

**Method**: `POST`

**Authentication**: Required (Supabase JWT)

**Purpose**: Generate AI-suggested follow-ups for a customer based on their history

### 5.2 Request Schema

```typescript
interface SuggestFollowUpsRequest {
  customer_id: string; // UUID
  max_suggestions?: number; // Default: 3, max: 5
  include_history?: boolean; // Whether to include full history in AI prompt
}
```

### 5.3 Customer History Schema

```typescript
interface CustomerHistory {
  customer: {
    id: string;
    name: string;
    type: string;
    status: string;
    lifecycle_stage: string;
    created_at: string;
  };
  recent_interactions: Array<{
    channel: string;
    direction: string;
    summary?: string;
    sentiment?: string;
    occurred_at: string;
  }>;
  pending_followups: Array<{
    title: string;
    due_at: string;
    priority: string;
  }>;
  tags: string[];
  // Future: work_orders, quotes (when modules are implemented)
}
```

### 5.4 Implementation: Edge Function

**File**: `supabase/functions/crm-ai-suggest-followups/index.ts`

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';
import { OpenAIProvider } from '../_shared/ai-provider.ts';

serve(async (req) => {
  try {
    const supabaseClient = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_ANON_KEY') ?? '',
      {
        global: {
          headers: { Authorization: req.headers.get('Authorization')! },
        },
      }
    );
    
    const {
      data: { user },
      error: authError,
    } = await supabaseClient.auth.getUser();
    
    if (authError || !user) {
      return new Response(
        JSON.stringify({ error: 'Unauthorized' }),
        { status: 401, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    const url = new URL(req.url);
    const customerId = url.pathname.split('/').pop();
    
    if (!customerId) {
      return new Response(
        JSON.stringify({ error: 'Customer ID is required' }),
        { status: 400, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    const body = await req.json();
    const maxSuggestions = Math.min(body.max_suggestions || 3, 5);
    
    // Get user's org_id
    const { data: profile } = await supabaseClient
      .from('profiles')
      .select('org_id, role')
      .eq('id', user.id)
      .single();
    
    // Check role (all roles except technician can request suggestions)
    if (profile.role === 'technician') {
      return new Response(
        JSON.stringify({ error: 'Technicians cannot request AI follow-up suggestions' }),
        { status: 403, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    // Check safeguards: prevent excessive suggestions
    const { data: recentSuggestions } = await supabaseClient
      .from('crm_followups')
      .select('id')
      .eq('customer_id', customerId)
      .eq('origin', 'ai_recommendation')
      .gte('created_at', new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString())
      .limit(10);
    
    if (recentSuggestions && recentSuggestions.length >= 10) {
      return new Response(
        JSON.stringify({ error: 'Too many AI suggestions in the last 7 days. Please wait before requesting more.' }),
        { status: 429, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    // Get customer history
    const customerHistory = await buildCustomerHistory(supabaseClient, customerId, profile.org_id);
    
    // Check rate limits
    const rateLimitCheck = await checkAIRateLimit(supabaseClient, profile.org_id);
    if (!rateLimitCheck.allowed) {
      return new Response(
        JSON.stringify({ error: 'AI rate limit exceeded', retry_after: rateLimitCheck.retry_after }),
        { status: 429, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    // Initialize AI provider
    const aiProvider = new OpenAIProvider(Deno.env.get('OPENAI_API_KEY') || '');
    
    // Get suggestions
    let suggestions;
    try {
      suggestions = await aiProvider.suggestFollowups(customerHistory);
      
      // Limit to max_suggestions
      if (suggestions.length > maxSuggestions) {
        suggestions = suggestions.slice(0, maxSuggestions);
      }
    } catch (error) {
      return new Response(
        JSON.stringify({ error: `AI processing failed: ${error.message}` }),
        { status: 500, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    // Create follow-ups
    const createdFollowups = [];
    const supabaseAdmin = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? '',
    );
    
    for (const suggestion of suggestions) {
      // Calculate due_at
      const dueAt = new Date();
      dueAt.setMinutes(dueAt.getMinutes() + suggestion.due_at_offset_minutes);
      
      // Check for duplicates (similar title and due date)
      const { data: existing } = await supabaseAdmin
        .from('crm_followups')
        .select('id')
        .eq('customer_id', customerId)
        .eq('title', suggestion.title)
        .eq('status', 'pending')
        .single();
      
      if (existing) {
        continue; // Skip duplicate
      }
      
      // Create follow-up
      const { data: followup, error: createError } = await supabaseAdmin.rpc(
        'crm_create_followup',
        {
          p_org_id: profile.org_id,
          p_customer_id: customerId,
          p_title: suggestion.title,
          p_description: `${suggestion.description}\n\nAI Reasoning: ${suggestion.reasoning}`,
          p_due_at: dueAt.toISOString(),
          p_priority: suggestion.priority,
          p_origin: 'ai_recommendation',
          p_created_by_user_id: user.id,
        }
      );
      
      if (!createError && followup) {
        createdFollowups.push(followup);
      }
    }
    
    // Log AI usage
    await logAIUsage(supabaseAdmin, profile.org_id, 'followup_suggestion', {
      customer_id: customerId,
      suggestions_count: suggestions.length,
      created_count: createdFollowups.length,
    });
    
    return new Response(
      JSON.stringify({
        customer_id: customerId,
        suggestions_requested: suggestions.length,
        followups_created: createdFollowups.length,
        followups: createdFollowups,
      }),
      { status: 200, headers: { 'Content-Type': 'application/json' } }
    );
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { 'Content-Type': 'application/json' } }
    );
  }
});

async function buildCustomerHistory(
  supabase: any,
  customerId: string,
  orgId: string
): Promise<CustomerHistory> {
  // Get customer
  const { data: customer } = await supabase
    .from('customers')
    .select('*')
    .eq('id', customerId)
    .eq('org_id', orgId)
    .single();
  
  // Get recent interactions (last 20)
  const { data: interactions } = await supabase
    .from('crm_interactions')
    .select('channel, direction, summary, sentiment, occurred_at')
    .eq('customer_id', customerId)
    .eq('org_id', orgId)
    .order('occurred_at', { ascending: false })
    .limit(20);
  
  // Get pending follow-ups
  const { data: followups } = await supabase
    .from('crm_followups')
    .select('title, due_at, priority')
    .eq('customer_id', customerId)
    .eq('org_id', orgId)
    .eq('status', 'pending')
    .order('due_at', { ascending: true })
    .limit(10);
  
  // Get tags
  const { data: tags } = await supabase
    .from('crm_customer_tags')
    .select('crm_tags(name)')
    .eq('customer_id', customerId)
    .eq('org_id', orgId);
  
  return {
    customer: {
      id: customer.id,
      name: customer.name,
      type: customer.type,
      status: customer.status,
      lifecycle_stage: customer.lifecycle_stage,
      created_at: customer.created_at,
    },
    recent_interactions: interactions || [],
    pending_followups: followups || [],
    tags: tags?.map((t: any) => t.crm_tags?.name).filter(Boolean) || [],
  };
}
```

---

## 6. Error Handling

### 6.1 AI Provider Errors

**Error Types**:
- `API_ERROR`: AI provider API error
- `TIMEOUT`: Request timeout
- `RATE_LIMIT`: Rate limit exceeded
- `INVALID_RESPONSE`: AI response cannot be parsed
- `QUOTA_EXCEEDED`: Monthly quota exceeded

**Handling Strategy**:
- Log all errors to `crm_ai_usage_log`
- Use fallback logic when possible (keyword-based sentiment, etc.)
- Return graceful error messages to users
- Do not block core functionality

### 6.2 Fallback Mechanisms

**Sentiment Analysis Fallback**:
- Keyword-based sentiment if AI fails
- Default to `neutral` if no text to analyze

**Segment Generation Fallback**:
- Return error message
- Do not create invalid segments

**Follow-Up Suggestions Fallback**:
- Return empty array if AI fails
- Log error for monitoring

---

## 7. Cost Management

### 7.1 Rate Limiting

**Per-Org Limits**:
- Segment generation: 10 per day
- Sentiment analysis: 1000 per day
- Follow-up suggestions: 100 per day

**Implementation**: Track in `crm_ai_usage_log` table

### 7.2 Cost Tracking

**Track**:
- Tokens used per request
- Estimated cost per request
- Daily/monthly totals per org

**Storage**: `crm_ai_usage_log.metadata` JSONB field

### 7.3 Optimization Strategies

- Use cheaper models when appropriate (GPT-3.5-turbo for sentiment)
- Cache results when possible
- Batch requests when feasible
- Limit data sent to AI (aggregated, anonymized)

---

## 8. Privacy & Security

### 8.1 Data Minimization

- Only send necessary fields to AI
- Aggregate data before sending
- Remove PII where possible
- Hash sensitive identifiers

### 8.2 Anonymization Strategy

**Before Sending to AI**:
- Remove: email addresses, phone numbers, full addresses
- Keep: customer IDs (for matching), aggregated metrics, anonymized tags
- Hash: customer names (optional, for additional privacy)

### 8.3 Audit Logging

- Log all AI API calls
- Track what data was sent
- Record AI responses (sanitized)
- Monitor for data leaks

---

## 9. Performance Considerations

### 9.1 Timeouts

- Segment generation: 60 seconds
- Sentiment analysis: 30 seconds
- Follow-up suggestions: 45 seconds

### 9.2 Async Processing

- Use queue system for batch operations
- Process sentiment analysis asynchronously
- Background jobs for large segment computations

### 9.3 Caching

- Cache AI results when appropriate
- Cache schema descriptions
- Cache aggregated customer data (TTL: 5 minutes)

---

## 10. Testing Requirements

### 10.1 Unit Tests

- AI provider abstraction layer
- Fallback logic
- Rate limiting logic
- Data anonymization functions

### 10.2 Integration Tests

- End-to-end AI segment generation
- Sentiment analysis flow
- Follow-up suggestion flow
- Error handling and fallbacks

### 10.3 Mock AI Provider

- Create mock provider for testing
- Simulate various AI responses
- Test error scenarios

---

## 11. Implementation Checklist

### Story CRM-033: AI-Driven Customer Segmentation
- [ ] Edge Function implemented (`POST /crm/segments/:id/compute-ai`)
- [ ] AI provider abstraction layer created
- [ ] Aggregated customer data function implemented
- [ ] Privacy controls (data minimization, anonymization)
- [ ] Rate limiting implemented
- [ ] Error handling and fallbacks
- [ ] AI usage logging
- [ ] Cost tracking
- [ ] Tests written
- [ ] Documentation with examples and limitations

### Story CRM-034: Sentiment Analysis for Interactions
- [ ] Edge Function implemented (`POST /crm/interactions/:id/analyze-sentiment`)
- [ ] Database trigger for automatic analysis
- [ ] Queue table and background processor
- [ ] Fallback sentiment analysis (keyword-based)
- [ ] Timeout handling
- [ ] Rate limiting
- [ ] Error handling (non-blocking)
- [ ] Tests written
- [ ] Documentation with costs and rate limits

### Story CRM-035: AI-Based Follow-Up Suggestions
- [ ] Edge Function implemented (`POST /crm/customers/:id/suggest-followups`)
- [ ] Customer history builder
- [ ] Duplicate detection
- [ ] Safeguards (max suggestions per period)
- [ ] Role-based authorization
- [ ] Follow-up creation integration
- [ ] Error handling
- [ ] Tests written
- [ ] Documentation with limitations

---

## 12. AI Provider Configuration

### 12.1 Environment Variables

```bash
# Required
OPENAI_API_KEY=sk-...

# Optional (for provider switching)
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_AI_API_KEY=...

# Configuration
AI_PROVIDER=openai # 'openai', 'anthropic', 'google'
AI_MODEL_SEGMENTATION=gpt-4-turbo-preview
AI_MODEL_SENTIMENT=gpt-3.5-turbo
AI_MODEL_FOLLOWUPS=gpt-4-turbo-preview
```

### 12.2 Provider Selection Logic

```typescript
function getAIProvider(): AIProvider {
  const provider = Deno.env.get('AI_PROVIDER') || 'openai';
  
  switch (provider) {
    case 'openai':
      return new OpenAIProvider(Deno.env.get('OPENAI_API_KEY') || '');
    case 'anthropic':
      return new AnthropicProvider(Deno.env.get('ANTHROPIC_API_KEY') || '');
    case 'google':
      return new GoogleAIProvider(Deno.env.get('GOOGLE_AI_API_KEY') || '');
    default:
      throw new Error(`Unsupported AI provider: ${provider}`);
  }
}
```

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 8 – AI & Analytics Features. All specifications are designed to be directly consumable by LLM-based code generators, with exact request/response schemas, validation rules, AI provider integration patterns, privacy controls, error handling, and cost management strategies defined.

