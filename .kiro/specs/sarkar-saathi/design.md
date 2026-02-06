# Design Document: Sarkar Saathi

## Overview

Sarkar Saathi is a voice-first AI assistant that bridges the digital divide for rural Indian citizens accessing government schemes. The system architecture emphasizes simplicity, reliability, and accessibility through a microservices approach that integrates voice telephony (Twilio), conversational AI (Claude API), translation services (Google Translate), and SMS notifications.

The design follows a layered architecture with clear separation between:
- **Presentation Layer**: Voice interface and optional web UI
- **Application Layer**: Conversational AI engine and business logic
- **Integration Layer**: External API adapters (Twilio, Claude, Google Translate)
- **Data Layer**: Scheme database and session management

Key design principles:
- **Voice-First**: All functionality accessible through voice interaction
- **Fail-Safe**: SMS fallback for connectivity issues
- **Stateless Services**: Horizontal scalability with session state externalized
- **Extensible Schema**: Easy addition of new government schemes
- **Privacy-Focused**: Minimal data retention, no persistent PII storage

## Architecture

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Layer                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Phone Call   │  │  SMS         │  │  Web UI      │          │
│  │  (Voice)     │  │  (Fallback)  │  │  (Demo)      │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
└─────────┼──────────────────┼──────────────────┼─────────────────┘
          │                  │                  │
┌─────────┼──────────────────┼──────────────────┼─────────────────┐
│         │    Integration Layer (API Gateway)  │                 │
│         ▼                  ▼                  ▼                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Twilio Voice │  │ Twilio SMS   │  │ REST API     │          │
│  │  Webhook     │  │  Webhook     │  │  Endpoints   │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
└─────────┼──────────────────┼──────────────────┼─────────────────┘
          │                  │                  │
┌─────────┼──────────────────┼──────────────────┼─────────────────┐
│         │      Application Layer (Node.js)    │                 │
│         ▼                  ▼                  ▼                 │
│  ┌────────────────────────────────────────────────┐             │
│  │         Conversation Manager                   │             │
│  │  - Session Management                          │             │
│  │  - Context Tracking                            │             │
│  │  - Language Preference                         │             │
│  └────────┬───────────────────────────────────────┘             │
│           │                                                     │
│  ┌────────┴───────────────────────────────────────┐             │
│  │         AI Engine (Claude API)                 │             │
│  │  - Intent Recognition                          │             │
│  │  - Entity Extraction                           │             │
│  │  - Response Generation                         │             │
│  └────────┬───────────────────────────────────────┘             │
│           │                                                     │
│  ┌────────┴───────────────────────────────────────┐             │
│  │         Business Logic Layer                   │             │
│  │  ┌──────────────────┐  ┌──────────────────┐   │             │
│  │  │ Eligibility      │  │ Scheme           │   │             │
│  │  │ Checker          │  │ Information      │   │             │
│  │  └──────────────────┘  └──────────────────┘   │             │
│  └────────┬───────────────────────────────────────┘             │
└───────────┼─────────────────────────────────────────────────────┘
            │
┌───────────┼─────────────────────────────────────────────────────┐
│           │         External Services Layer                     │
│           ▼                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Google       │  │ Twilio       │  │ Claude API   │          │
│  │ Translate    │  │ Services     │  │              │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
            │
┌───────────┼─────────────────────────────────────────────────────┐
│           │         Data Layer                                  │
│           ▼                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Scheme       │  │ Session      │  │ Interaction  │          │
│  │ Database     │  │ Store        │  │ Logs         │          │
│  │ (MongoDB)    │  │ (Redis)      │  │ (MongoDB)    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

**Primary Voice Interaction Flow:**

1. **User Initiates Call** → Twilio receives call → Webhook triggers Node.js backend
2. **Session Creation** → Generate session ID → Store in Redis with 10-minute TTL
3. **Language Selection** → Play language menu → Capture user selection → Store preference
4. **Voice Input** → Twilio captures speech → Converts to text (STT)
5. **Translation** → If Regional_Language, translate to English via Google Translate
6. **AI Processing** → Send to Claude API with conversation context → Extract intent and entities
7. **Business Logic** → Route to appropriate handler (eligibility check, scheme info, etc.)
8. **Eligibility Check** (if applicable):
   - Collect required fields through conversational prompts
   - Validate against scheme criteria in database
   - Generate eligibility result
9. **Response Generation** → Claude API generates natural language response
10. **Translation** → Translate response to user's language
11. **Text-to-Speech** → Twilio converts text to speech → Plays to user
12. **SMS Notification** → Send summary via Twilio SMS
13. **Session Update** → Update context in Redis
14. **Loop** → Continue conversation or end call

**SMS Fallback Flow:**

1. **User Sends SMS** → Twilio receives SMS → Webhook triggers backend
2. **Text Processing** → Extract intent from SMS text
3. **Translation & AI** → Same as voice flow (steps 5-9)
4. **SMS Response** → Send response via Twilio SMS (160 char chunks)
5. **Follow-up** → Handle multi-turn conversation via SMS thread

## Components and Interfaces

### 1. Voice Interface Service

**Responsibilities:**
- Handle Twilio voice webhooks
- Manage call flow and IVR logic
- Convert speech to text and text to speech
- Handle DTMF input for fallback navigation

**Key Interfaces:**

```typescript
interface VoiceService {
  handleIncomingCall(callSid: string, from: string): Promise<TwiMLResponse>;
  handleSpeechInput(callSid: string, speechResult: string, confidence: number): Promise<TwiMLResponse>;
  handleGatherComplete(callSid: string, digits: string): Promise<TwiMLResponse>;
  synthesizeSpeech(text: string, language: string): Promise<AudioStream>;
}

interface TwiMLResponse {
  say?: { text: string; language: string; voice: string };
  gather?: { input: string[]; timeout: number; action: string };
  redirect?: string;
  hangup?: boolean;
}
```

**Technology:** Node.js with Twilio SDK, Express.js for webhook endpoints

### 2. Conversation Manager

**Responsibilities:**
- Maintain session state and conversation context
- Track user language preference
- Manage conversation flow and turn-taking
- Handle session expiry and cleanup

**Key Interfaces:**

```typescript
interface ConversationManager {
  createSession(userId: string, language: string): Promise<Session>;
  getSession(sessionId: string): Promise<Session | null>;
  updateSession(sessionId: string, updates: Partial<Session>): Promise<void>;
  addMessage(sessionId: string, role: 'user' | 'assistant', content: string): Promise<void>;
  getConversationHistory(sessionId: string): Promise<Message[]>;
  expireSession(sessionId: string): Promise<void>;
}

interface Session {
  id: string;
  userId: string;
  language: string;
  createdAt: Date;
  lastActivity: Date;
  context: ConversationContext;
  messages: Message[];
}

interface ConversationContext {
  currentIntent?: string;
  collectedData: Record<string, any>;
  currentScheme?: string;
  eligibilityStatus?: 'pending' | 'eligible' | 'ineligible';
}

interface Message {
  role: 'user' | 'assistant';
  content: string;
  timestamp: Date;
  language: string;
}
```

**Technology:** Node.js service with Redis for session storage

### 3. AI Engine Service

**Responsibilities:**
- Process user queries using Claude API
- Extract intents and entities
- Generate contextually appropriate responses
- Handle multi-turn conversations

**Key Interfaces:**

```typescript
interface AIEngine {
  processQuery(
    query: string,
    context: ConversationContext,
    history: Message[]
  ): Promise<AIResponse>;
  
  extractIntent(query: string): Promise<Intent>;
  extractEntities(query: string, intent: Intent): Promise<Entity[]>;
  generateResponse(intent: Intent, data: any, language: string): Promise<string>;
}

interface AIResponse {
  intent: Intent;
  entities: Entity[];
  response: string;
  confidence: number;
  requiresFollowUp: boolean;
  nextAction?: string;
}

interface Intent {
  name: string;
  confidence: number;
  category: 'eligibility_check' | 'scheme_info' | 'application_help' | 'general_query';
}

interface Entity {
  type: string;
  value: string;
  confidence: number;
}
```

**Technology:** Node.js with Anthropic Claude API SDK

**Prompt Engineering Strategy:**

```typescript
const SYSTEM_PROMPT = `You are Sarkar Saathi, a helpful voice assistant for rural Indian citizens.
Your role is to help users understand and access government schemes, particularly PM-KISAN.

Guidelines:
- Use simple, clear language appropriate for low-literacy users
- Be patient and ask one question at a time
- Confirm understanding before proceeding
- Provide step-by-step guidance
- Be culturally sensitive and respectful
- When checking eligibility, collect: land ownership, land size, farmer category, state
- Explain eligibility results clearly with reasons

Current conversation language: {language}
Current scheme context: {scheme}`;
```

### 4. Translation Service

**Responsibilities:**
- Translate between English and regional languages
- Maintain translation quality and context
- Handle translation errors gracefully

**Key Interfaces:**

```typescript
interface TranslationService {
  translate(text: string, from: string, to: string): Promise<string>;
  detectLanguage(text: string): Promise<string>;
  getSupportedLanguages(): string[];
}

interface TranslationCache {
  get(key: string): Promise<string | null>;
  set(key: string, value: string, ttl: number): Promise<void>;
}
```

**Technology:** Google Translate API with Redis caching for common phrases

**Supported Languages (MVP):**
- English (en)
- Hindi (hi)
- Marathi (mr)

### 5. Eligibility Checker Service

**Responsibilities:**
- Validate user information against scheme criteria
- Determine eligibility status
- Provide detailed eligibility explanations
- Handle scheme-specific validation rules

**Key Interfaces:**

```typescript
interface EligibilityChecker {
  checkEligibility(scheme: string, userData: UserData): Promise<EligibilityResult>;
  getRequiredFields(scheme: string): Promise<string[]>;
  validateField(scheme: string, field: string, value: any): Promise<ValidationResult>;
}

interface UserData {
  landOwnership: 'owned' | 'leased' | 'none';
  landSizeHectares?: number;
  farmerCategory: 'small' | 'marginal' | 'large';
  state: string;
  hasAadhaar?: boolean;
  hasBankAccount?: boolean;
}

interface EligibilityResult {
  eligible: boolean;
  reasons: string[];
  nextSteps?: string[];
  requiredDocuments?: string[];
  estimatedBenefit?: string;
}

interface ValidationResult {
  valid: boolean;
  error?: string;
  suggestion?: string;
}
```

**PM-KISAN Eligibility Rules:**

```typescript
const PM_KISAN_RULES = {
  landOwnership: {
    required: true,
    validValues: ['owned'],
    errorMessage: 'PM-KISAN is only for land-owning farmers'
  },
  landSizeHectares: {
    required: true,
    max: 2.0,
    errorMessage: 'PM-KISAN is for small and marginal farmers with up to 2 hectares'
  },
  farmerCategory: {
    required: true,
    validValues: ['small', 'marginal'],
    errorMessage: 'Only small and marginal farmers are eligible'
  },
  exclusions: [
    'institutional_landholders',
    'government_employees',
    'income_tax_payers',
    'professionals'
  ]
};
```

### 6. Scheme Information Service

**Responsibilities:**
- Retrieve scheme details from database
- Provide scheme descriptions and benefits
- List available schemes
- Support scheme search and filtering

**Key Interfaces:**

```typescript
interface SchemeService {
  getScheme(schemeId: string): Promise<Scheme>;
  listSchemes(filters?: SchemeFilters): Promise<Scheme[]>;
  searchSchemes(query: string): Promise<Scheme[]>;
  getSchemesByCategory(category: string): Promise<Scheme[]>;
}

interface Scheme {
  id: string;
  name: string;
  nameTranslations: Record<string, string>;
  description: string;
  descriptionTranslations: Record<string, string>;
  category: string;
  eligibilityCriteria: EligibilityCriteria;
  benefits: string[];
  applicationProcess: string[];
  requiredDocuments: string[];
  officialWebsite?: string;
  helplineNumber?: string;
}

interface EligibilityCriteria {
  rules: Rule[];
  requiredFields: string[];
}

interface Rule {
  field: string;
  operator: 'equals' | 'lessThan' | 'greaterThan' | 'in' | 'notIn';
  value: any;
  errorMessage: string;
}
```

### 7. SMS Gateway Service

**Responsibilities:**
- Send SMS notifications
- Handle SMS-based interactions
- Manage SMS conversation threads
- Handle delivery failures and retries

**Key Interfaces:**

```typescript
interface SMSService {
  sendSMS(to: string, message: string, language: string): Promise<SMSResult>;
  handleIncomingSMS(from: string, body: string): Promise<void>;
  sendEligibilityResult(to: string, result: EligibilityResult, language: string): Promise<void>;
  retryFailedSMS(messageId: string): Promise<SMSResult>;
}

interface SMSResult {
  messageId: string;
  status: 'sent' | 'delivered' | 'failed';
  error?: string;
}
```

**SMS Message Templates:**

```typescript
const SMS_TEMPLATES = {
  eligibilityConfirmed: {
    hi: 'बधाई हो! आप PM-KISAN योजना के लिए पात्र हैं। लाभ: ₹6000/वर्ष। अगले कदम: {nextSteps}',
    mr: 'अभिनंदन! तुम्ही PM-KISAN योजनेसाठी पात्र आहात. लाभ: ₹6000/वर्ष. पुढील पायऱ्या: {nextSteps}',
    en: 'Congratulations! You are eligible for PM-KISAN. Benefit: ₹6000/year. Next steps: {nextSteps}'
  },
  eligibilityDenied: {
    hi: 'क्षमा करें, आप PM-KISAN के लिए पात्र नहीं हैं। कारण: {reasons}',
    mr: 'क्षमस्व, तुम्ही PM-KISAN साठी पात्र नाही. कारण: {reasons}',
    en: 'Sorry, you are not eligible for PM-KISAN. Reason: {reasons}'
  }
};
```

### 8. Web Interface (Demo/Admin)

**Responsibilities:**
- Provide demo interface for hackathon presentation
- Display conversation flow visualization
- Admin panel for scheme management
- Analytics dashboard

**Key Components:**

```typescript
// React Components
interface WebUIComponents {
  ConversationDemo: React.FC<{ sessionId: string }>;
  SchemeExplorer: React.FC;
  EligibilitySimulator: React.FC;
  AnalyticsDashboard: React.FC;
  AdminPanel: React.FC;
}

// REST API Endpoints
interface WebAPI {
  'POST /api/demo/start-conversation': (language: string) => Promise<{ sessionId: string }>;
  'POST /api/demo/send-message': (sessionId: string, message: string) => Promise<{ response: string }>;
  'GET /api/schemes': () => Promise<Scheme[]>;
  'GET /api/schemes/:id': (id: string) => Promise<Scheme>;
  'POST /api/eligibility/check': (schemeId: string, userData: UserData) => Promise<EligibilityResult>;
  'GET /api/analytics/summary': () => Promise<AnalyticsSummary>;
}
```

**Technology:** React with TypeScript, Tailwind CSS for styling, mobile-first responsive design

## Data Models

### Database Schema (MongoDB)

**Schemes Collection:**

```typescript
{
  _id: ObjectId,
  schemeId: string,  // "pm-kisan"
  name: string,
  nameTranslations: {
    hi: string,
    mr: string,
    en: string
  },
  description: string,
  descriptionTranslations: {
    hi: string,
    mr: string,
    en: string
  },
  category: string,  // "agriculture", "health", "housing"
  eligibilityCriteria: {
    rules: [
      {
        field: string,
        operator: string,
        value: any,
        errorMessage: string,
        errorMessageTranslations: {
          hi: string,
          mr: string,
          en: string
        }
      }
    ],
    requiredFields: string[]
  },
  benefits: string[],
  benefitTranslations: {
    hi: string[],
    mr: string[],
    en: string[]
  },
  applicationProcess: string[],
  applicationProcessTranslations: {
    hi: string[],
    mr: string[],
    en: string[]
  },
  requiredDocuments: string[],
  requiredDocumentsTranslations: {
    hi: string[],
    mr: string[],
    en: string[]
  },
  officialWebsite: string,
  helplineNumber: string,
  active: boolean,
  createdAt: Date,
  updatedAt: Date
}
```

**Interaction Logs Collection:**

```typescript
{
  _id: ObjectId,
  sessionId: string,
  userId: string,  // phone number (hashed)
  channel: 'voice' | 'sms' | 'web',
  language: string,
  startTime: Date,
  endTime: Date,
  messages: [
    {
      role: 'user' | 'assistant',
      content: string,
      originalLanguage: string,
      translatedContent: string,
      timestamp: Date,
      intent: string,
      confidence: number
    }
  ],
  schemeQueried: string,
  eligibilityChecked: boolean,
  eligibilityResult: {
    eligible: boolean,
    scheme: string,
    reasons: string[]
  },
  smsNotificationSent: boolean,
  userSatisfaction: number,  // 1-5 rating if collected
  errors: [
    {
      type: string,
      message: string,
      timestamp: Date
    }
  ],
  metadata: {
    callDuration: number,
    apiCallsCount: number,
    translationCallsCount: number,
    responseTimeAvg: number
  }
}
```

### Session Store Schema (Redis)

```typescript
// Key: session:{sessionId}
// TTL: 600 seconds (10 minutes)
{
  sessionId: string,
  userId: string,
  language: string,
  channel: 'voice' | 'sms' | 'web',
  createdAt: number,  // Unix timestamp
  lastActivity: number,
  context: {
    currentIntent: string,
    currentScheme: string,
    collectedData: {
      landOwnership: string,
      landSizeHectares: number,
      farmerCategory: string,
      state: string,
      // ... other collected fields
    },
    eligibilityStatus: 'pending' | 'eligible' | 'ineligible',
    conversationStage: 'greeting' | 'collecting_info' | 'checking_eligibility' | 'providing_results' | 'followup'
  },
  messages: [
    {
      role: 'user' | 'assistant',
      content: string,
      timestamp: number
    }
  ]
}
```

### Translation Cache Schema (Redis)

```typescript
// Key: translation:{hash(text)}:{from}:{to}
// TTL: 86400 seconds (24 hours)
{
  originalText: string,
  translatedText: string,
  fromLanguage: string,
  toLanguage: string,
  cachedAt: number
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Language Support Validation

*For any* voice input or text input, the system should accept inputs in Hindi or Marathi and reject inputs in unsupported languages with an appropriate error message.

**Validates: Requirements 1.1, 2.1**

### Property 2: System Performance Bounds

*For any* user query under normal conditions, the system should generate a response within 3 seconds, and speech-to-text conversion should complete within 2 seconds.

**Validates: Requirements 1.2, 4.4, 8.1, 8.4**

### Property 3: Text-to-Speech Language Consistency

*For any* system response, when converted to speech, the audio output should be in the same language as the user's selected language preference.

**Validates: Requirements 1.3**

### Property 4: Low Confidence Handling

*For any* voice input with confidence score below threshold (e.g., 0.6), the system should request the user to repeat their input rather than proceeding with potentially incorrect interpretation.

**Validates: Requirements 1.4**

### Property 5: Session State Persistence

*For any* active user session, the language preference and collected user data should remain consistent across all interactions within that session until session expiry.

**Validates: Requirements 2.2, 5.3, 7.2**

### Property 6: Required Field Collection

*For any* eligibility check request, the system should collect all required fields for the specific scheme through conversational prompts before performing validation.

**Validates: Requirements 3.1**

### Property 7: Eligibility Validation Correctness

*For any* user data provided, the eligibility checker should validate the information against the scheme's defined rules and return a result that correctly reflects whether all criteria are met.

**Validates: Requirements 3.2**

### Property 8: Eligibility Response Completeness

*For any* eligibility check result, the system response should include the eligibility status (eligible/ineligible), specific reasons for the determination, and appropriate next steps or required documents when eligible.

**Validates: Requirements 3.3, 3.4**

### Property 9: Clarification for Ambiguous Data

*For any* user input that contains ambiguous or incomplete information required for eligibility checking, the system should request additional clarifying information before proceeding with validation.

**Validates: Requirements 3.5**

### Property 10: Intent Recognition and Entity Extraction

*For any* user query, the AI engine should extract at least one intent with a confidence score and identify relevant entities mentioned in the query.

**Validates: Requirements 4.1**

### Property 11: Follow-up Question Generation

*For any* conversation state where required information is missing, the AI engine should generate a follow-up question to collect the missing data.

**Validates: Requirements 4.2**

### Property 12: SMS Notification After Eligibility Check

*For any* completed eligibility check, the system should send an SMS notification to the user's phone number containing a summary of the eligibility result in the user's preferred language.

**Validates: Requirements 5.1, 5.3**

### Property 13: SMS Retry Logic

*For any* failed SMS send attempt, the system should retry up to 3 times with exponential backoff (e.g., 1s, 2s, 4s delays) before marking the SMS as permanently failed.

**Validates: Requirements 5.4**

### Property 14: Scheme Data Currency

*For any* scheme query, the system should retrieve and return the most current information from the database, reflecting any updates made to scheme details.

**Validates: Requirements 6.2, 6.3**

### Property 15: Extensible Scheme Schema

*For any* new scheme added to the database with different eligibility criteria structures, the system should store and retrieve the scheme correctly without requiring code changes.

**Validates: Requirements 6.4, 13.1**

### Property 16: Unique Session Creation

*For any* user initiation of contact, the system should create a new session with a unique session identifier that doesn't collide with existing active sessions.

**Validates: Requirements 7.1**

### Property 17: Session Expiry and Cleanup

*For any* user session that has been inactive for 10 minutes, the system should expire the session and clear all temporary data from the session store.

**Validates: Requirements 7.3**

### Property 18: New Session After Expiry

*For any* user who returns after their session has expired, the system should create a new session rather than attempting to resume the expired session.

**Validates: Requirements 7.4**

### Property 19: Data Privacy and Retention

*For any* user session, sensitive personally identifiable information should not be persisted beyond the session lifetime, and any logged interactions should have PII anonymized.

**Validates: Requirements 7.5, 12.3, 12.4**

### Property 20: Responsive Web Design

*For any* screen width between 320px and 768px, the web interface should render correctly with all elements visible and accessible without horizontal scrolling.

**Validates: Requirements 9.2**

### Property 21: Data Usage Minimization

*For any* typical user interaction (eligibility check with 5-10 message exchanges), the total data transferred should be minimized to accommodate users with limited data plans (target: <500KB per session).

**Validates: Requirements 9.4**

### Property 22: Technical Term Explanations

*For any* system response containing technical terms (e.g., "Aadhaar", "hectare", "marginal farmer"), the response should include a simple explanation of the term appropriate for low-literacy users.

**Validates: Requirements 10.3**

### Property 23: Error Message Localization and Clarity

*For any* error that occurs during system operation, the error message presented to the user should be in the user's selected language and use simple, non-technical language.

**Validates: Requirements 11.1**

### Property 24: External Service Fallback

*For any* external service failure (translation, AI, SMS), the system should detect the failure and activate appropriate fallback mechanisms (e.g., use cached translations, switch to SMS mode, use default language).

**Validates: Requirements 11.2**

### Property 25: Comprehensive Error Logging

*For any* error that occurs, the system should log the error with timestamp, session ID, error type, and sufficient context for debugging, while ensuring no PII is included in logs.

**Validates: Requirements 11.5**

### Property 26: Data Minimization in Collection

*For any* scheme eligibility check, the system should request only the fields marked as required in the scheme's configuration and not collect additional unnecessary information.

**Validates: Requirements 12.1**

### Property 27: Dynamic Conversation Flow Adaptation

*For any* scheme selected by the user, the AI engine should adapt the conversation flow to collect the scheme-specific required fields and apply the scheme-specific validation rules.

**Validates: Requirements 13.3, 13.4**

### Property 28: Scheme Discovery and Recommendation

*For any* user profile with collected demographic information, when multiple schemes are available, the system should suggest schemes that match the user's profile characteristics.

**Validates: Requirements 13.5**

### Property 29: External API Failure Detection

*For any* external API call that fails or times out, the system should detect the failure within 5 seconds and either retry with backoff or activate fallback mechanisms.

**Validates: Requirements 14.4**

### Property 30: API Rate Limiting and Retry

*For any* external API integration, the system should implement rate limiting to stay within API quotas and retry failed requests with exponential backoff before returning an error to the user.

**Validates: Requirements 14.5**

### Property 31: Comprehensive Interaction Logging

*For any* user interaction, the system should log the interaction with timestamp, session ID, user message, system response, intent, confidence score, and any errors encountered.

**Validates: Requirements 15.1, 15.2, 15.3**



## Error Handling

### Error Categories and Handling Strategies

**1. External Service Failures**

- **Twilio Voice/SMS Failures**:
  - Detection: Monitor webhook response codes and Twilio status callbacks
  - Fallback: Switch to SMS mode if voice fails; log and alert if SMS fails
  - User Communication: "We're having trouble with the phone connection. We'll send you the information via SMS."

- **Claude API Failures**:
  - Detection: Timeout after 5 seconds, monitor HTTP status codes
  - Fallback: Use rule-based intent detection for common queries; offer human support escalation
  - Retry: 3 attempts with exponential backoff (1s, 2s, 4s)
  - User Communication: "I'm having trouble processing your request. Let me try again."

- **Google Translate API Failures**:
  - Detection: Timeout after 3 seconds, monitor HTTP status codes
  - Fallback: Use cached translations for common phrases; fall back to Hindi as default
  - User Communication: "I'll continue in Hindi for now."

**2. Voice Recognition Errors**

- **Low Confidence Scores (<0.6)**:
  - Action: Request user to repeat input
  - User Communication: "I didn't catch that clearly. Could you please repeat?"
  - Limit: After 3 failed attempts, offer SMS fallback

- **Unsupported Language Detection**:
  - Action: Inform user of supported languages, request language selection
  - User Communication: "I can help you in Hindi or Marathi. Which language would you prefer?"

- **Background Noise/Unclear Audio**:
  - Action: Request user to move to quieter location or speak more clearly
  - User Communication: "There's some background noise. Could you try speaking in a quieter place?"

**3. Data Validation Errors**

- **Invalid Field Values**:
  - Action: Explain the valid range/options, request correct input
  - Example: "Land size should be in hectares. For example, if you have 5 bighas, that's about 1.5 hectares."

- **Missing Required Fields**:
  - Action: Ask conversational follow-up questions to collect missing data
  - Example: "To check your eligibility, I need to know: do you own the land or lease it?"

- **Inconsistent Data**:
  - Action: Point out the inconsistency, ask for clarification
  - Example: "You mentioned you're a large farmer, but PM-KISAN is for small and marginal farmers with up to 2 hectares. How much land do you have?"

**4. Session Management Errors**

- **Session Expiry During Interaction**:
  - Action: Create new session, briefly summarize what was discussed, continue from last known state if possible
  - User Communication: "Let's continue. You were asking about PM-KISAN eligibility."

- **Session Store Unavailable (Redis Down)**:
  - Action: Use in-memory session storage as temporary fallback, log critical error
  - Impact: Sessions won't persist across server restarts, but immediate functionality maintained

**5. Database Errors**

- **Scheme Data Not Found**:
  - Action: Log error, inform user, offer to help with other schemes
  - User Communication: "I don't have information about that scheme right now. Can I help you with PM-KISAN or another scheme?"

- **Database Connection Failure**:
  - Action: Retry connection 3 times, use cached scheme data if available, alert administrators
  - User Communication: "I'm having trouble accessing the latest information. Please try again in a few minutes."

**6. Performance Degradation**

- **Response Time >3 Seconds**:
  - Action: Send acknowledgment message to user, continue processing
  - User Communication: "Let me check that for you. This might take a moment."

- **High Load (>100 Concurrent Sessions)**:
  - Action: Enable request queuing, scale horizontally if on cloud infrastructure
  - Monitoring: Alert administrators when approaching capacity

### Error Response Format

All errors returned to users follow this structure:

```typescript
interface UserError {
  message: string;  // User-friendly message in user's language
  suggestedAction?: string;  // What the user should do next
  fallbackOption?: string;  // Alternative way to proceed
  supportContact?: string;  // Helpline number if escalation needed
}
```

### Logging Strategy

**Error Logs:**
```typescript
{
  timestamp: Date,
  sessionId: string,
  userId: string,  // hashed phone number
  errorType: string,
  errorMessage: string,
  stackTrace: string,
  context: {
    currentIntent: string,
    conversationStage: string,
    lastUserMessage: string  // sanitized
  },
  severity: 'low' | 'medium' | 'high' | 'critical',
  resolved: boolean
}
```

**Severity Levels:**
- **Low**: Single user affected, automatic recovery possible (e.g., translation cache miss)
- **Medium**: Single user affected, manual intervention may be needed (e.g., repeated voice recognition failures)
- **High**: Multiple users affected or data integrity concern (e.g., database connection issues)
- **Critical**: System-wide failure or security concern (e.g., external API completely down, data breach attempt)

## Testing Strategy

### Dual Testing Approach

The testing strategy combines **unit tests** for specific examples and edge cases with **property-based tests** for universal correctness properties. Both approaches are complementary and necessary for comprehensive coverage.

**Unit Tests** focus on:
- Specific examples that demonstrate correct behavior
- Edge cases and boundary conditions
- Error handling scenarios
- Integration points between components

**Property-Based Tests** focus on:
- Universal properties that hold for all inputs
- Comprehensive input coverage through randomization
- Invariants that must be maintained
- Round-trip properties (e.g., serialization/deserialization)

### Property-Based Testing Configuration

**Library Selection:** 
- **JavaScript/TypeScript**: fast-check
- **Reason**: Mature library with excellent TypeScript support, configurable generators, shrinking capabilities

**Test Configuration:**
- Minimum 100 iterations per property test (due to randomization)
- Each property test must reference its design document property
- Tag format: `// Feature: sarkar-saathi, Property {number}: {property_text}`

**Example Property Test Structure:**

```typescript
import fc from 'fast-check';

describe('Sarkar Saathi Property Tests', () => {
  // Feature: sarkar-saathi, Property 7: Eligibility Validation Correctness
  it('should correctly validate user data against PM-KISAN rules', () => {
    fc.assert(
      fc.property(
        fc.record({
          landOwnership: fc.constantFrom('owned', 'leased', 'none'),
          landSizeHectares: fc.float({ min: 0, max: 10 }),
          farmerCategory: fc.constantFrom('small', 'marginal', 'large'),
          state: fc.constantFrom('Maharashtra', 'Punjab', 'Kerala')
        }),
        (userData) => {
          const result = checkEligibility('pm-kisan', userData);
          
          // Property: Result should match manual rule evaluation
          const expectedEligible = 
            userData.landOwnership === 'owned' &&
            userData.landSizeHectares <= 2.0 &&
            (userData.farmerCategory === 'small' || userData.farmerCategory === 'marginal');
          
          expect(result.eligible).toBe(expectedEligible);
          
          // Property: Reasons should be provided
          expect(result.reasons.length).toBeGreaterThan(0);
          
          // Property: If eligible, next steps should be provided
          if (result.eligible) {
            expect(result.nextSteps).toBeDefined();
            expect(result.nextSteps.length).toBeGreaterThan(0);
          }
        }
      ),
      { numRuns: 100 }
    );
  });
});
```

### Test Coverage by Component

**1. Voice Interface Service**
- Unit Tests:
  - Twilio webhook handling with valid/invalid payloads
  - TwiML response generation for different scenarios
  - DTMF input handling
- Property Tests:
  - Property 3: TTS language consistency across random responses
  - Property 4: Low confidence handling for various confidence scores

**2. Conversation Manager**
- Unit Tests:
  - Session creation and retrieval
  - Session expiry edge cases (exactly at 10 minutes, just before, just after)
  - Message history management
- Property Tests:
  - Property 5: Session state persistence across multiple interactions
  - Property 16: Unique session ID generation
  - Property 17: Session expiry after timeout
  - Property 18: New session creation after expiry

**3. AI Engine Service**
- Unit Tests:
  - Intent recognition for specific example queries
  - Entity extraction for known patterns
  - Error handling when Claude API returns unexpected responses
- Property Tests:
  - Property 10: Intent and entity extraction for random queries
  - Property 11: Follow-up question generation when data is missing

**4. Translation Service**
- Unit Tests:
  - Translation of specific phrases
  - Cache hit/miss scenarios
  - Fallback to Hindi when translation fails
- Property Tests:
  - Property 5: Language preference maintained across session (includes translation)

**5. Eligibility Checker Service**
- Unit Tests:
  - PM-KISAN eligibility for specific farmer profiles
  - Edge cases: exactly 2 hectares, 0 hectares, negative values
  - Missing required fields
- Property Tests:
  - Property 6: Required field collection completeness
  - Property 7: Eligibility validation correctness
  - Property 8: Eligibility response completeness
  - Property 9: Clarification requests for ambiguous data
  - Property 26: Data minimization in collection

**6. Scheme Information Service**
- Unit Tests:
  - Scheme retrieval by ID
  - Scheme search with specific queries
  - Handling of non-existent schemes
- Property Tests:
  - Property 14: Scheme data currency after updates
  - Property 15: Extensible schema for new schemes

**7. SMS Gateway Service**
- Unit Tests:
  - SMS sending with specific messages
  - SMS template rendering with variables
  - Delivery failure handling
- Property Tests:
  - Property 12: SMS notification after eligibility check
  - Property 13: SMS retry logic with exponential backoff

**8. Web Interface**
- Unit Tests:
  - Component rendering with specific props
  - API endpoint responses
  - Form validation
- Property Tests:
  - Property 20: Responsive design across screen widths

### Integration Testing

**End-to-End Scenarios:**
1. Complete eligibility check flow (voice → AI → eligibility → SMS)
2. SMS fallback when voice fails
3. Multi-turn conversation with context maintenance
4. Language switching mid-conversation
5. Session expiry and recovery

**Integration Test Tools:**
- Supertest for API endpoint testing
- Twilio test credentials for webhook simulation
- Mock external APIs (Claude, Google Translate) for deterministic testing

### Performance Testing

**Load Testing:**
- Tool: Artillery or k6
- Scenarios:
  - 100 concurrent voice sessions
  - 500 concurrent SMS interactions
  - Mixed load (voice + SMS + web)
- Metrics:
  - Response time (p50, p95, p99)
  - Error rate
  - Throughput (requests/second)

**Stress Testing:**
- Gradually increase load until system degrades
- Identify bottlenecks (database, external APIs, session store)
- Verify graceful degradation and recovery

### Manual Testing

**Usability Testing:**
- Test with actual users from target demographic
- Observe interaction patterns and pain points
- Collect feedback on voice clarity, conversation flow, language quality

**Accessibility Testing:**
- Test with users of varying literacy levels
- Verify voice-only interaction completeness
- Test with different regional accents

**Device Testing:**
- Test on various phone models (basic phones, smartphones)
- Test on different network conditions (2G, 3G, 4G, WiFi)
- Test web interface on different mobile browsers

### Continuous Integration

**CI Pipeline:**
1. Lint and format check (ESLint, Prettier)
2. Unit tests (Jest)
3. Property-based tests (fast-check)
4. Integration tests
5. Build and package
6. Deploy to staging environment
7. Run smoke tests on staging

**Test Execution:**
- All tests run on every commit
- Property tests run with 100 iterations in CI
- Extended property tests (1000+ iterations) run nightly

### Test Data Management

**Scheme Test Data:**
- Maintain test schemes with known eligibility rules
- Include edge cases in test data (boundary values, special characters)
- Separate test database from production

**User Test Data:**
- Generate synthetic user data for testing
- Use anonymized real data patterns (with consent)
- Never use actual PII in tests

**Translation Test Data:**
- Maintain corpus of common phrases in all supported languages
- Include edge cases (very long text, special characters, numbers)
- Verify translation quality manually for test corpus

## Deployment Architecture

### Infrastructure

**Hosting:** AWS (or similar cloud provider)

**Components:**
- **Application Servers**: EC2 instances or ECS containers running Node.js
- **Load Balancer**: Application Load Balancer for distributing traffic
- **Database**: MongoDB Atlas (managed service)
- **Session Store**: ElastiCache Redis (managed service)
- **File Storage**: S3 for logs and static assets
- **CDN**: CloudFront for web interface assets

**Scaling Strategy:**
- Horizontal scaling for application servers (auto-scaling based on CPU/memory)
- Vertical scaling for database (upgrade instance size as needed)
- Redis cluster for session store high availability

### Environment Configuration

**Development:**
- Local MongoDB and Redis instances
- Twilio test credentials
- Mock external APIs for faster development

**Staging:**
- Mirrors production infrastructure at smaller scale
- Uses test Twilio phone numbers
- Separate databases and session stores

**Production:**
- Full infrastructure with redundancy
- Production Twilio phone numbers
- Monitoring and alerting enabled
- Automated backups

### Deployment Process

1. Code review and approval
2. Merge to main branch
3. CI pipeline runs all tests
4. Build Docker image
5. Deploy to staging
6. Run smoke tests on staging
7. Manual approval for production deployment
8. Blue-green deployment to production
9. Monitor metrics for 30 minutes
10. Rollback if issues detected

### Monitoring and Alerting

**Metrics to Monitor:**
- Request rate and response time
- Error rate by type
- External API latency and error rate
- Database query performance
- Session store hit rate
- Active session count
- SMS delivery rate

**Alerting Thresholds:**
- Error rate >5% for 5 minutes
- Response time p95 >5 seconds for 5 minutes
- External API error rate >10% for 3 minutes
- Database connection pool exhausted
- Redis memory usage >80%
- Active sessions >80% of capacity

**Monitoring Tools:**
- CloudWatch for AWS metrics
- Custom application metrics (Prometheus + Grafana)
- Log aggregation (CloudWatch Logs or ELK stack)
- Uptime monitoring (Pingdom or UptimeRobot)

### Security Considerations

**Data Security:**
- TLS 1.2+ for all external communication
- Encryption at rest for database
- No PII in logs (hash phone numbers, redact sensitive fields)
- Regular security audits

**API Security:**
- API keys stored in environment variables or secrets manager
- Rate limiting on all endpoints
- Input validation and sanitization
- CORS configuration for web interface

**Access Control:**
- IAM roles for AWS resources
- Principle of least privilege
- MFA for production access
- Audit logs for all administrative actions

### Backup and Disaster Recovery

**Backup Strategy:**
- Daily automated database backups (retained for 30 days)
- Point-in-time recovery enabled
- Session store snapshots (retained for 7 days)
- Configuration and code in version control

**Disaster Recovery:**
- RTO (Recovery Time Objective): 4 hours
- RPO (Recovery Point Objective): 24 hours
- Multi-AZ deployment for high availability
- Documented recovery procedures
- Quarterly disaster recovery drills

### Cost Optimization

**Estimated Monthly Costs (MVP):**
- EC2/ECS: $100-200 (2-4 instances)
- MongoDB Atlas: $50-100 (shared cluster)
- ElastiCache Redis: $50-100 (cache.t3.micro)
- Twilio: $0.0085/minute voice + $0.0075/SMS (usage-based)
- Claude API: $0.008/1K tokens (usage-based)
- Google Translate: $20/1M characters (usage-based)
- Total: ~$300-500/month + usage-based costs

**Optimization Strategies:**
- Use reserved instances for predictable workloads
- Implement caching aggressively (translations, scheme data)
- Optimize API calls (batch requests, reduce redundant calls)
- Use spot instances for non-critical workloads
- Monitor and eliminate unused resources
