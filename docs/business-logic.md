# 🏥 Cancer Support App - Business Logic Documentation

## 📋 Table of Contents
1. [System Overview](#system-overview)
2. [User Management](#user-management)
3. [Authentication & Authorization](#authentication--authorization)
4. [Community Features](#community-features)
5. [Medical Records Management](#medical-records-management)
6. [AI Reminders System](#ai-reminders-system)
7. [Diet & Exercise Planning](#diet--exercise-planning)
8. [Data Analytics & Insights](#data-analytics--insights)
9. [Monetization Logic](#monetization-logic)
10. [Security & Compliance](#security--compliance)

---

## 🎯 System Overview

### Core Purpose
A digital health platform that empowers cancer patients through community-driven emotional support, AI-assisted care reminders, and secure medical record management.

### Technology Stack
- **Frontend**: Next.js 14 with TypeScript, Tailwind CSS
- **Backend**: Golang with Gin Framework
- **API Architecture**: Next.js API routes call Golang backend (Server-Side-Rendering)
- **Database**: PostgreSQL with GORM
- **File Storage**: AWS S3 or similar cloud storage
- **Authentication**: JWT with Golang middleware, managed through Next.js server-side
- **Real-time**: WebSocket/Server-Sent Events for chat and notifications
- **AI Integration**: OpenAI API or similar for intelligent suggestions
- **API Documentation**: Swagger/OpenAPI 3.0

---

## � API Architecture & Communication Pattern

### Next.js Server-Side to Golang Backend Pattern

The application follows a **Server-Side Rendering (SSR)** pattern where:

1. **Client Browser** → **Next.js Frontend**
2. **Next.js API Routes** → **Golang Backend APIs**
3. **Golang Backend** → **PostgreSQL Database**

#### Next.js API Routes Structure
```typescript
// pages/api/auth/login.ts
import type { NextApiRequest, NextApiResponse } from 'next';
import { backendAPI } from '../../../lib/backend-client';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'POST') {
    return res.status(405).json({ message: 'Method not allowed' });
  }

  try {
    // Call Golang backend from Next.js server-side
    const response = await backendAPI.post('/api/v1/auth/login', req.body);
    
    // Set HTTP-only cookie for JWT token (security best practice)
    res.setHeader('Set-Cookie', `auth-token=${response.data.token}; HttpOnly; Path=/; SameSite=Strict`);
    
    res.status(200).json({ user: response.data.user });
  } catch (error) {
    res.status(401).json({ message: 'Authentication failed' });
  }
}

// pages/api/community/posts.ts
import type { NextApiRequest, NextApiResponse } from 'next';
import { authenticatedBackendRequest } from '../../../lib/backend-client';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  try {
    // Extract JWT from HTTP-only cookie
    const token = extractTokenFromCookie(req.headers.cookie);
    
    if (req.method === 'GET') {
      const response = await authenticatedBackendRequest(token, 'GET', '/api/v1/community/posts');
      return res.status(200).json(response.data);
    }
    
    if (req.method === 'POST') {
      const response = await authenticatedBackendRequest(token, 'POST', '/api/v1/community/posts', req.body);
      return res.status(201).json(response.data);
    }
    
    res.status(405).json({ message: 'Method not allowed' });
  } catch (error) {
    res.status(500).json({ message: 'Server error' });
  }
}
```

#### Backend HTTP Client Configuration
```typescript
// lib/backend-client.ts
import axios from 'axios';

const GOLANG_BACKEND_URL = process.env.GOLANG_BACKEND_URL || 'http://localhost:8080';

export const backendAPI = axios.create({
  baseURL: GOLANG_BACKEND_URL,
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});

export async function authenticatedBackendRequest(
  token: string,
  method: 'GET' | 'POST' | 'PUT' | 'DELETE',
  endpoint: string,
  data?: any
) {
  const config = {
    method,
    url: endpoint,
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    ...(data && { data }),
  };

  return await backendAPI.request(config);
}

export function extractTokenFromCookie(cookieString?: string): string | null {
  if (!cookieString) return null;
  
  const match = cookieString.match(/auth-token=([^;]+)/);
  return match ? match[1] : null;
}
```

#### Authentication Flow (Server-Side)
```typescript
// lib/auth.ts - Server-side authentication utilities
import jwt from 'jsonwebtoken';
import { backendAPI } from './backend-client';

export interface AuthUser {
  id: string;
  email: string;
  role: 'patient' | 'doctor' | 'caregiver';
  subscriptionTier: 'free' | 'premium';
}

export async function validateServerSideAuth(req: NextApiRequest): Promise<AuthUser | null> {
  try {
    const token = extractTokenFromCookie(req.headers.cookie);
    if (!token) return null;

    // Verify token with Golang backend
    const response = await authenticatedBackendRequest(token, 'GET', '/api/v1/auth/verify');
    return response.data.user;
  } catch (error) {
    return null;
  }
}

export function requireAuth(handler: NextApiHandler) {
  return async (req: NextApiRequest, res: NextApiResponse) => {
    const user = await validateServerSideAuth(req);
    
    if (!user) {
      return res.status(401).json({ message: 'Unauthorized' });
    }

    // Add user to request object
    (req as any).user = user;
    return handler(req, res);
  };
}

// Usage in API routes
export default requireAuth(async function handler(req: NextApiRequest, res: NextApiResponse) {
  const user = (req as any).user;
  // Protected route logic here
});
```

#### Frontend Data Fetching (Client-Side)
```typescript
// lib/api.ts - Client-side API calls to Next.js routes
export async function fetchCommunityPosts() {
  const response = await fetch('/api/community/posts', {
    method: 'GET',
    credentials: 'include', // Include HTTP-only cookies
  });
  
  if (!response.ok) {
    throw new Error('Failed to fetch posts');
  }
  
  return response.json();
}

export async function createCommunityPost(postData: CreatePostRequest) {
  const response = await fetch('/api/community/posts', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    credentials: 'include',
    body: JSON.stringify(postData),
  });
  
  if (!response.ok) {
    throw new Error('Failed to create post');
  }
  
  return response.json();
}

export async function loginUser(credentials: LoginCredentials) {
  const response = await fetch('/api/auth/login', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    credentials: 'include',
    body: JSON.stringify(credentials),
  });
  
  if (!response.ok) {
    throw new Error('Login failed');
  }
  
  return response.json();
}
```

#### Next.js Pages with Server-Side Props
```typescript
// pages/dashboard.tsx
import { GetServerSideProps } from 'next';
import { validateServerSideAuth } from '../lib/auth';
import { authenticatedBackendRequest } from '../lib/backend-client';

interface DashboardProps {
  user: AuthUser;
  reminders: Reminder[];
  recentPosts: CommunityPost[];
}

export default function Dashboard({ user, reminders, recentPosts }: DashboardProps) {
  return (
    <div className="dashboard">
      <h1>Welcome back, {user.firstName}!</h1>
      
      <section className="reminders">
        <h2>Today's Reminders</h2>
        {reminders.map(reminder => (
          <ReminderCard key={reminder.id} reminder={reminder} />
        ))}
      </section>
      
      <section className="community">
        <h2>Recent Community Posts</h2>
        {recentPosts.map(post => (
          <PostCard key={post.id} post={post} />
        ))}
      </section>
    </div>
  );
}

export const getServerSideProps: GetServerSideProps = async (context) => {
  // Validate authentication on server-side
  const user = await validateServerSideAuth(context.req);
  
  if (!user) {
    return {
      redirect: {
        destination: '/login',
        permanent: false,
      },
    };
  }

  try {
    const token = extractTokenFromCookie(context.req.headers.cookie);
    
    // Fetch data from Golang backend server-side
    const [remindersResponse, postsResponse] = await Promise.all([
      authenticatedBackendRequest(token, 'GET', '/api/v1/reminders'),
      authenticatedBackendRequest(token, 'GET', '/api/v1/community/posts?limit=10'),
    ]);

    return {
      props: {
        user,
        reminders: remindersResponse.data,
        recentPosts: postsResponse.data,
      },
    };
  } catch (error) {
    return {
      props: {
        user,
        reminders: [],
        recentPosts: [],
      },
    };
  }
};
```

### Benefits of This Architecture

#### 1. **Security Advantages**
- **HTTP-Only Cookies**: JWT tokens stored in HTTP-only cookies, preventing XSS attacks
- **Server-Side Validation**: All authentication happens on Next.js server, never exposing tokens to client
- **CORS Protection**: Direct backend access blocked from browser
- **Environment Isolation**: Backend URL and secrets only available on server

#### 2. **Performance Benefits**
- **Server-Side Rendering**: Initial page load includes data, improving perceived performance
- **Request Optimization**: Can batch multiple backend calls in single Next.js API route
- **Caching Strategies**: Implement caching at Next.js level before hitting Golang backend
- **Reduced Client Bundle**: No HTTP client libraries needed on frontend

#### 3. **Development Advantages**
- **Type Safety**: Full TypeScript support across Next.js and Golang integration
- **Error Handling**: Centralized error handling in Next.js API routes
- **Logging & Monitoring**: Request logging at Next.js layer before backend
- **Testing**: Easier to mock Next.js API routes than direct backend calls

#### 4. **Scalability Considerations**
- **Load Balancing**: Multiple Golang backend instances behind Next.js
- **Rate Limiting**: Implement at Next.js level to protect Golang backend
- **Request Transformation**: Data transformation and validation in Next.js layer
- **API Versioning**: Version management through Next.js API routes

---

### User Types & Roles

#### 1. Patient (Primary User)
```typescript
interface Patient {
  id: string;
  email: string;
  profile: {
    firstName: string;
    lastName: string;
    dateOfBirth: Date;
    gender: 'male' | 'female' | 'other';
    cancerType: string;
    diagnosisDate: Date;
    treatmentStage: 'newly_diagnosed' | 'in_treatment' | 'remission' | 'survivor';
    location: {
      city: string;
      state: string;
      country: string;
    };
    isPublic: boolean; // Privacy setting
  };
  subscriptionTier: 'free' | 'premium';
  preferences: {
    language: string;
    timezone: string;
    notifications: NotificationSettings;
  };
  createdAt: Date;
  lastActive: Date;
}
```

#### 2. Doctor/Healthcare Provider
```typescript
interface Doctor {
  id: string;
  email: string;
  profile: {
    firstName: string;
    lastName: string;
    specialization: string;
    licenseNumber: string;
    hospitalAffiliation: string;
    experience: number;
  };
  verificationStatus: 'pending' | 'verified' | 'rejected';
  patients: string[]; // Patient IDs
  institutionId?: string;
}
```

#### 3. Caregiver/Family Member
```typescript
interface Caregiver {
  id: string;
  email: string;
  profile: {
    firstName: string;
    lastName: string;
    relationship: 'spouse' | 'parent' | 'child' | 'sibling' | 'friend' | 'other';
  };
  connectedPatients: string[]; // Patient IDs (with patient consent)
  accessLevel: 'view_only' | 'manage_reminders' | 'full_access';
}
```

### User Registration Flow
1. **Email/Phone Verification**
2. **Role Selection** (Patient/Doctor/Caregiver)
3. **Profile Setup** (based on role)
4. **Privacy Agreement** (HIPAA/GDPR compliance)
5. **Onboarding Tutorial**
6. **Optional: Connect with healthcare provider**

---

## 🔐 Authentication & Authorization

### Authentication Methods
- Email/Password
- Google OAuth
- Phone Number OTP
- Biometric (mobile app)

### Role-Based Access Control (RBAC)

#### Permissions Matrix
| Feature | Patient | Doctor | Caregiver | Admin |
|---------|---------|--------|-----------|-------|
| View Own Profile | ✅ | ✅ | ✅ | ✅ |
| Edit Own Profile | ✅ | ✅ | ✅ | ✅ |
| View Community Posts | ✅ | ❌ | ❌ | ✅ |
| Create Community Posts | ✅ | ❌ | ❌ | ✅ |
| View Medical Records | ✅ | ✅* | ✅* | ✅ |
| Upload Medical Records | ✅ | ✅* | ✅* | ✅ |
| Set Reminders | ✅ | ✅* | ✅* | ✅ |
| Access AI Insights | ✅ | ✅ | ❌ | ✅ |

*With explicit patient consent

### Data Access Levels
```typescript
enum AccessLevel {
  PUBLIC = 'public',           // Community posts, public profiles
  PRIVATE = 'private',         // Personal data, medical records
  SHARED = 'shared',          // Shared with specific users (doctor/caregiver)
  ANONYMOUS = 'anonymous'      // Analytics data (anonymized)
}
```

---

## 🤝 Community Features

### Community Management System

#### Community Types & Structure
```typescript
interface Community {
  id: string;
  name: string;
  description: string;
  type: 'public' | 'private' | 'expert_led' | 'regional';
  category: 'cancer_type' | 'treatment_stage' | 'age_group' | 'interest' | 'location';
  tags: string[];
  memberCount: number;
  adminId: string;
  moderatorIds: string[];
  createdAt: Date;
  settings: {
    requireApproval: boolean;
    allowVideoMeets: boolean;
    allowDirectMessaging: boolean;
    contentModerationLevel: 'strict' | 'moderate' | 'relaxed';
  };
}
```

#### Video Calling Integration
```typescript
interface VideoSession {
  id: string;
  communityId?: string;
  type: 'group_community' | 'one_on_one' | 'expert_session';
  title: string;
  description?: string;
  scheduledTime: Date;
  duration: number; // minutes
  maxParticipants?: number;
  hostId: string;
  participants: {
    userId: string;
    joinedAt?: Date;
    leftAt?: Date;
    status: 'invited' | 'accepted' | 'declined' | 'joined' | 'left';
  }[];
  googleMeetLink: string;
  linkExpiration: Date;
  isRecurring: boolean;
  recurringPattern?: string;
  status: 'scheduled' | 'active' | 'completed' | 'cancelled';
  createdAt: Date;
}
```

#### Chat System Architecture
```typescript
interface ChatMessage {
  id: string;
  conversationId: string;
  senderId: string;
  messageType: 'text' | 'image' | 'document' | 'voice' | 'system';
  content: string;
  attachments?: {
    type: 'image' | 'document' | 'voice';
    url: string;
    filename: string;
    size: number;
  }[];
  mentions?: string[]; // User IDs mentioned in message
  reactions: {
    userId: string;
    emoji: string;
    timestamp: Date;
  }[];
  isEdited: boolean;
  editedAt?: Date;
  replyToMessageId?: string;
  timestamp: Date;
  deliveryStatus: 'sent' | 'delivered' | 'read';
  isEncrypted: boolean;
}

interface Conversation {
  id: string;
  type: 'direct' | 'group' | 'community';
  participants: string[];
  communityId?: string;
  title?: string;
  lastMessage?: ChatMessage;
  unreadCount: Record<string, number>; // userId -> unread count
  createdAt: Date;
  updatedAt: Date;
  settings: {
    allowFileSharing: boolean;
    allowVoiceMessages: boolean;
    messageRetentionDays: number;
  };
}
```

### Post Types
```typescript
interface CommunityPost {
  id: string;
  authorId: string;
  type: 'story' | 'question' | 'milestone' | 'support' | 'resource';
  title: string;
  content: string;
  tags: string[];
  attachments?: {
    type: 'image' | 'document';
    url: string;
    filename: string;
  }[];
  privacy: 'public' | 'supporters_only' | 'anonymous';
  likes: number;
  comments: Comment[];
  createdAt: Date;
  updatedAt: Date;
  isInspirational: boolean;
  moderationStatus: 'pending' | 'approved' | 'flagged' | 'removed';
}
```

### Community Features Logic

#### 1. Story Sharing
- **Success Stories**: Milestone celebrations, treatment completions
- **Daily Experiences**: Sharing daily struggles and victories
- **Resource Sharing**: Helpful tips, treatment centers, doctors
- **Anonymous Option**: Allow posting without revealing identity

#### 2. Support System
- **Peer Matching**: Connect patients with similar cancer types/stages
- **Mentorship Program**: Survivors mentoring newly diagnosed patients
- **Support Groups**: Virtual groups based on cancer type, age, location
- **Crisis Support**: Emergency emotional support system

#### 3. Content Moderation
```typescript
interface ModerationRule {
  trigger: string;
  action: 'flag' | 'auto_remove' | 'review';
  reason: string;
}

const moderationRules: ModerationRule[] = [
  {
    trigger: 'medical_advice_keywords',
    action: 'flag',
    reason: 'Potential medical advice from non-professional'
  },
  {
    trigger: 'spam_patterns',
    action: 'auto_remove',
    reason: 'Spam content detected'
  },
  {
    trigger: 'sensitive_content',
    action: 'review',
    reason: 'May contain sensitive information'
  }
];
```

---

## 📋 Medical Records Management

### Record Types
```typescript
interface MedicalRecord {
  id: string;
  patientId: string;
  type: 'prescription' | 'lab_report' | 'scan_result' | 'doctor_note' | 'insurance_doc';
  title: string;
  description?: string;
  fileUrl: string;
  filename: string;
  fileSize: number;
  mimeType: string;
  uploadedBy: string; // User ID
  uploadedAt: Date;
  tags: string[];
  category: string;
  isEncrypted: boolean;
  accessLevel: AccessLevel;
  sharedWith: {
    userId: string;
    role: 'doctor' | 'caregiver';
    permissions: 'view' | 'download' | 'edit';
    sharedAt: Date;
  }[];
  metadata: {
    scanDate?: Date;
    reportDate?: Date;
    doctorName?: string;
    hospitalName?: string;
    testResults?: Record<string, any>;
  };
}
```

### File Management Logic
1. **Upload Process**:
   - File validation (type, size, virus scan)
   - Client-side encryption before upload
   - Server-side storage with additional encryption
   - Automatic OCR for text extraction and search

2. **Organization System**:
   - Auto-categorization using AI
   - Tag suggestions based on content
   - Timeline view of medical history
   - Smart search across all documents

3. **Sharing Logic**:
   - Temporary sharing links with expiration
   - Granular permission controls
   - Activity logging for shared documents
   - Revoke access functionality

---

## ⏰ AI Reminders System

### Reminder Types
```typescript
interface Reminder {
  id: string;
  patientId: string;
  type: 'medication' | 'appointment' | 'water' | 'food' | 'exercise' | 'custom';
  title: string;
  description?: string;
  schedule: {
    frequency: 'once' | 'daily' | 'weekly' | 'monthly' | 'custom';
    time: string; // HH:mm format
    days?: number[]; // 0-6 (Sunday-Saturday)
    interval?: number; // For custom frequencies
    customPattern?: string; // Cron-like pattern
  };
  isActive: boolean;
  createdAt: Date;
  nextTrigger: Date;
  metadata: {
    medicationName?: string;
    dosage?: string;
    waterAmount?: number; // in ml
    exerciseType?: string;
    appointmentDetails?: {
      doctorName: string;
      location: string;
      phone: string;
    };
  };
  preferences: {
    snoozeEnabled: boolean;
    snoozeTime: number; // minutes
    escalationEnabled: boolean;
    escalationContacts: string[]; // Caregiver IDs
  };
}
```

### AI Intelligence Features

#### 1. Smart Scheduling
```typescript
interface SmartScheduling {
  analyzePatternFromPrescription(prescriptionText: string): Reminder[];
  suggestOptimalTimes(patientId: string, reminderType: string): string[];
  detectConflicts(newReminder: Reminder, existingReminders: Reminder[]): Conflict[];
  adaptToUserBehavior(patientId: string, reminderHistory: ReminderLog[]): void;
}
```

#### 2. Intelligent Notifications
- **Adaptive Timing**: Learn user's response patterns
- **Context Awareness**: Avoid notifications during sleep hours
- **Escalation Logic**: Notify caregivers if critical reminders are missed
- **Motivation Messages**: Personalized encouragement based on progress

#### 3. Adherence Tracking
```typescript
interface AdherenceMetrics {
  medicationCompliance: number; // Percentage
  waterIntakeGoals: number;
  exerciseFrequency: number;
  appointmentAttendance: number;
  trendsOverTime: {
    date: Date;
    complianceScore: number;
  }[];
}
```

---

## 🥗 Diet & Exercise Planning

### Nutrition Management
```typescript
interface NutritionPlan {
  id: string;
  patientId: string;
  createdBy: string; // Nutritionist/AI system
  type: 'ai_generated' | 'expert_created' | 'custom';
  cancerType: string;
  treatmentPhase: string;
  restrictions: string[]; // Allergies, intolerances
  goals: {
    calories: number;
    protein: number;
    carbs: number;
    fats: number;
    fiber: number;
    hydration: number; // ml per day
  };
  meals: {
    breakfast: FoodItem[];
    lunch: FoodItem[];
    dinner: FoodItem[];
    snacks: FoodItem[];
  };
  supplements: {
    name: string;
    dosage: string;
    timing: string;
    purpose: string;
  }[];
  validFrom: Date;
  validTo: Date;
  isActive: boolean;
}

interface FoodItem {
  name: string;
  quantity: string;
  calories: number;
  nutrients: Record<string, number>;
  benefits: string[];
  alternativeOptions: string[];
}
```

### Exercise Planning
```typescript
interface ExercisePlan {
  id: string;
  patientId: string;
  level: 'beginner' | 'intermediate' | 'advanced';
  adaptedFor: string[]; // Treatment side effects, physical limitations
  weeklySchedule: {
    monday?: ExerciseSession;
    tuesday?: ExerciseSession;
    wednesday?: ExerciseSession;
    thursday?: ExerciseSession;
    friday?: ExerciseSession;
    saturday?: ExerciseSession;
    sunday?: ExerciseSession;
  };
  goals: {
    endurance: number;
    strength: number;
    flexibility: number;
    mentalWellbeing: number;
  };
  progressTracking: {
    date: Date;
    completed: boolean;
    duration: number;
    intensity: number;
    notes: string;
  }[];
}

interface ExerciseSession {
  type: 'cardio' | 'strength' | 'flexibility' | 'meditation' | 'breathing';
  duration: number; // minutes
  intensity: 'low' | 'moderate' | 'high';
  exercises: {
    name: string;
    sets?: number;
    reps?: number;
    duration?: number;
    instructions: string;
    modifications: string[]; // For different ability levels
  }[];
}
```

---

## 📊 Data Analytics & Insights

### Patient Dashboard Metrics
```typescript
interface PatientInsights {
  healthScore: number; // 0-100 overall health metric
  adherenceScore: number; // Medication/treatment compliance
  communityEngagement: number; // Social interaction level
  progressIndicators: {
    symptomTrends: {
      symptom: string;
      severity: number;
      date: Date;
    }[];
    energyLevels: {
      level: number;
      date: Date;
    }[];
    moodTracking: {
      mood: number;
      date: Date;
    }[];
  };
  appointmentHistory: {
    date: Date;
    doctor: string;
    type: string;
    attended: boolean;
    notes?: string;
  }[];
  milestones: {
    date: Date;
    type: 'treatment_completed' | 'clear_scan' | 'anniversary' | 'custom';
    description: string;
    isShared: boolean;
  }[];
}
```

### AI-Generated Insights
1. **Pattern Recognition**: Identify correlations between symptoms, treatments, and lifestyle
2. **Predictive Analytics**: Forecast potential complications or improvements
3. **Personalized Recommendations**: Suggest diet, exercise, or lifestyle changes
4. **Risk Assessment**: Alert for concerning patterns requiring medical attention

---

## 💰 Monetization Logic

### Subscription Tiers

#### Free Tier
```typescript
interface FreeTierLimits {
  storageLimit: 100; // MB
  reminderLimit: 10;
  communityAccess: true;
  basicInsights: true;
  aiChatLimit: 5; // per day
  supportAccess: 'community_only';
}
```

#### Premium Tier (₹199/month or ₹1,999/year)
```typescript
interface PremiumTierFeatures {
  storageLimit: 'unlimited';
  reminderLimit: 'unlimited';
  expertDietPlans: true;
  advancedInsights: true;
  aiChatLimit: 'unlimited';
  prioritySupport: true;
  telehealth: true;
  familyAccess: 3; // Number of caregiver accounts
  exportData: true;
  customization: true;
}
```

### Revenue Tracking
```typescript
interface RevenueMetrics {
  subscriptionRevenue: {
    monthly: number;
    yearly: number;
    lifetime: number;
  };
  institutionalLicenses: {
    hospitalPartnerships: number;
    ngoPartnerships: number;
    clinicLicenses: number;
  };
  apiUsage: {
    calls: number;
    revenue: number;
  };
  donations: {
    crowdfunding: number;
    directDonations: number;
    corporateSponsorship: number;
  };
}
```

### Payment Processing
```typescript
interface PaymentFlow {
  paymentMethods: ('card' | 'upi' | 'wallet' | 'bank_transfer')[];
  subscriptionManagement: {
    autoRenewal: boolean;
    gracePeriod: number; // days
    downgradeBehavior: 'immediate' | 'end_of_cycle';
  };
  refundPolicy: {
    window: number; // days
    proRatedRefunds: boolean;
  };
}
```

---

## 🔒 Security & Compliance

### Data Encryption
```typescript
interface SecurityMeasures {
  encryption: {
    inTransit: 'TLS 1.3';
    atRest: 'AES-256';
    endToEnd: boolean; // For sensitive medical data
  };
  accessControl: {
    mfa: boolean;
    sessionTimeout: number; // minutes
    ipWhitelisting: boolean;
    deviceManagement: boolean;
  };
  auditLogging: {
    dataAccess: boolean;
    userActions: boolean;
    systemEvents: boolean;
    retentionPeriod: number; // days
  };
}
```

### Privacy Controls
```typescript
interface PrivacySettings {
  dataRetention: {
    userRequested: boolean; // Right to be forgotten
    automaticCleanup: number; // days for inactive accounts
    backupRetention: number; // days
  };
  dataSharing: {
    researchOptIn: boolean;
    anonymizedAnalytics: boolean;
    thirdPartyIntegrations: string[];
  };
  consentManagement: {
    granularConsent: boolean;
    consentVersion: string;
    withdrawalProcess: 'immediate' | 'scheduled';
  };
}
```

### Compliance Framework
1. **HIPAA Compliance** (US market)
2. **GDPR Compliance** (EU market)
3. **DISHA Guidelines** (India market)
4. **ISO 27001** (Information Security)
5. **SOC 2 Type II** (Service Organization Controls)

---

## 🚀 Implementation Phases

### Phase 1: MVP (Months 1-3)
- User registration and authentication
- Basic community features (post, comment, like)
- File upload and basic organization
- Simple reminder system
- Basic mobile-responsive web app

### Phase 2: Enhanced Features (Months 4-6)
- AI-powered reminders and suggestions
- Advanced diet planning
- Enhanced security and encryption
- Mobile app (React Native/Flutter)
- Payment integration

### Phase 3: Intelligence & Integration (Months 7-9)
- Advanced AI insights and analytics
- Healthcare provider portal
- API for third-party integrations
- Telemedicine features
- Advanced reporting

### Phase 4: Scale & Global (Months 10-12)
- Multi-language support
- Global expansion features
- Wearable device integration
- Advanced AI models
- Enterprise features

---

## 📱 Technical Architecture

### Backend Architecture (Golang + Gin)

#### Project Structure
```
backend/
├── cmd/
│   └── server/
│       └── main.go                 // Application entry point
├── internal/
│   ├── config/
│   │   └── config.go              // Configuration management
│   ├── handlers/
│   │   ├── auth.go                // Authentication handlers
│   │   ├── users.go               // User management handlers
│   │   ├── community.go           // Community features handlers
│   │   ├── medical.go             // Medical records handlers
│   │   ├── reminders.go           // Reminder system handlers
│   │   └── payments.go            // Payment processing handlers
│   ├── middleware/
│   │   ├── auth.go                // JWT authentication middleware
│   │   ├── cors.go                // CORS middleware
│   │   ├── logging.go             // Request logging middleware
│   │   └── ratelimit.go           // Rate limiting middleware
│   ├── models/
│   │   ├── user.go                // User-related models
│   │   ├── community.go           // Community models
│   │   ├── medical.go             // Medical record models
│   │   └── reminder.go            // Reminder models
│   ├── services/
│   │   ├── auth_service.go        // Authentication business logic
│   │   ├── user_service.go        // User management logic
│   │   ├── community_service.go   // Community features logic
│   │   ├── medical_service.go     // Medical records logic
│   │   ├── reminder_service.go    // Reminder system logic
│   │   └── ai_service.go          // AI integration logic
│   ├── repository/
│   │   ├── user_repository.go     // User data access layer
│   │   ├── community_repository.go// Community data access
│   │   └── medical_repository.go  // Medical records data access
│   └── utils/
│       ├── encryption.go          // Encryption utilities
│       ├── jwt.go                 // JWT utilities
│       └── validation.go          // Input validation
├── pkg/
│   ├── database/
│   │   └── connection.go          // Database connection
│   ├── storage/
│   │   └── s3.go                  // File storage utilities
│   └── logger/
│       └── logger.go              // Logging utilities
├── migrations/
│   └── *.sql                      // Database migration files
├── docs/
│   └── swagger.yaml               // API documentation
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── go.mod
├── go.sum
└── README.md
```

#### Core Golang Models

##### User Models
```go
package models

import (
    "time"
    "gorm.io/gorm"
)

type User struct {
    ID        uint           `json:"id" gorm:"primaryKey"`
    Email     string         `json:"email" gorm:"uniqueIndex;not null"`
    Password  string         `json:"-" gorm:"not null"`
    Role      UserRole       `json:"role" gorm:"not null"`
    IsActive  bool           `json:"is_active" gorm:"default:true"`
    CreatedAt time.Time      `json:"created_at"`
    UpdatedAt time.Time      `json:"updated_at"`
    DeletedAt gorm.DeletedAt `json:"-" gorm:"index"`
    
    // Polymorphic association
    Patient   *Patient   `json:"patient,omitempty" gorm:"foreignKey:UserID"`
    Doctor    *Doctor    `json:"doctor,omitempty" gorm:"foreignKey:UserID"`
    Caregiver *Caregiver `json:"caregiver,omitempty" gorm:"foreignKey:UserID"`
}

type UserRole string

const (
    RolePatient   UserRole = "patient"
    RoleDoctor    UserRole = "doctor"
    RoleCaregiver UserRole = "caregiver"
    RoleAdmin     UserRole = "admin"
)

type Patient struct {
    ID                uint                 `json:"id" gorm:"primaryKey"`
    UserID            uint                 `json:"user_id" gorm:"uniqueIndex"`
    FirstName         string               `json:"first_name" gorm:"not null"`
    LastName          string               `json:"last_name" gorm:"not null"`
    DateOfBirth       time.Time            `json:"date_of_birth"`
    Gender            string               `json:"gender" gorm:"type:varchar(10)"`
    CancerType        string               `json:"cancer_type"`
    DiagnosisDate     time.Time            `json:"diagnosis_date"`
    TreatmentStage    TreatmentStage       `json:"treatment_stage"`
    City              string               `json:"city"`
    State             string               `json:"state"`
    Country           string               `json:"country"`
    IsPublic          bool                 `json:"is_public" gorm:"default:false"`
    SubscriptionTier  SubscriptionTier     `json:"subscription_tier" gorm:"default:'free'"`
    Language          string               `json:"language" gorm:"default:'en'"`
    Timezone          string               `json:"timezone" gorm:"default:'UTC'"`
    CreatedAt         time.Time            `json:"created_at"`
    UpdatedAt         time.Time            `json:"updated_at"`
    
    // Relationships
    MedicalRecords    []MedicalRecord      `json:"medical_records,omitempty"`
    Reminders         []Reminder           `json:"reminders,omitempty"`
    CommunityPosts    []CommunityPost      `json:"community_posts,omitempty"`
}

type TreatmentStage string

const (
    NewlyDiagnosed TreatmentStage = "newly_diagnosed"
    InTreatment    TreatmentStage = "in_treatment"
    Remission      TreatmentStage = "remission"
    Survivor       TreatmentStage = "survivor"
)

type SubscriptionTier string

const (
    FreeTier    SubscriptionTier = "free"
    PremiumTier SubscriptionTier = "premium"
)
```

##### Medical Records Models
```go
type MedicalRecord struct {
    ID           uint        `json:"id" gorm:"primaryKey"`
    PatientID    uint        `json:"patient_id" gorm:"not null"`
    Type         RecordType  `json:"type" gorm:"not null"`
    Title        string      `json:"title" gorm:"not null"`
    Description  string      `json:"description"`
    FileURL      string      `json:"file_url" gorm:"not null"`
    FileName     string      `json:"file_name" gorm:"not null"`
    FileSize     int64       `json:"file_size"`
    MimeType     string      `json:"mime_type"`
    UploadedBy   uint        `json:"uploaded_by"`
    Tags         []string    `json:"tags" gorm:"type:text[]"`
    Category     string      `json:"category"`
    IsEncrypted  bool        `json:"is_encrypted" gorm:"default:true"`
    AccessLevel  AccessLevel `json:"access_level" gorm:"default:'private'"`
    ScanDate     *time.Time  `json:"scan_date,omitempty"`
    ReportDate   *time.Time  `json:"report_date,omitempty"`
    DoctorName   string      `json:"doctor_name"`
    HospitalName string      `json:"hospital_name"`
    CreatedAt    time.Time   `json:"created_at"`
    UpdatedAt    time.Time   `json:"updated_at"`
    
    // Relationships
    Patient      Patient     `json:"patient" gorm:"foreignKey:PatientID"`
    SharedWith   []RecordShare `json:"shared_with,omitempty"`
}

type RecordType string

const (
    Prescription RecordType = "prescription"
    LabReport    RecordType = "lab_report"
    ScanResult   RecordType = "scan_result"
    DoctorNote   RecordType = "doctor_note"
    InsuranceDoc RecordType = "insurance_doc"
)

type AccessLevel string

const (
    PublicAccess    AccessLevel = "public"
    PrivateAccess   AccessLevel = "private"
    SharedAccess    AccessLevel = "shared"
    AnonymousAccess AccessLevel = "anonymous"
)
```

##### Reminder System Models
```go
type Reminder struct {
    ID               uint            `json:"id" gorm:"primaryKey"`
    PatientID        uint            `json:"patient_id" gorm:"not null"`
    Type             ReminderType    `json:"type" gorm:"not null"`
    Title            string          `json:"title" gorm:"not null"`
    Description      string          `json:"description"`
    Frequency        Frequency       `json:"frequency" gorm:"not null"`
    Time             string          `json:"time" gorm:"not null"` // HH:MM format
    Days             []int           `json:"days" gorm:"type:integer[]"`
    IsActive         bool            `json:"is_active" gorm:"default:true"`
    NextTrigger      time.Time       `json:"next_trigger"`
    MedicationName   string          `json:"medication_name,omitempty"`
    Dosage           string          `json:"dosage,omitempty"`
    WaterAmount      int             `json:"water_amount,omitempty"`
    ExerciseType     string          `json:"exercise_type,omitempty"`
    SnoozeEnabled    bool            `json:"snooze_enabled" gorm:"default:true"`
    SnoozeTime       int             `json:"snooze_time" gorm:"default:15"` // minutes
    CreatedAt        time.Time       `json:"created_at"`
    UpdatedAt        time.Time       `json:"updated_at"`
    
    // Relationships
    Patient          Patient         `json:"patient" gorm:"foreignKey:PatientID"`
    ReminderLogs     []ReminderLog   `json:"reminder_logs,omitempty"`
}

type ReminderType string

const (
    MedicationReminder ReminderType = "medication"
    AppointmentReminder ReminderType = "appointment"
    WaterReminder      ReminderType = "water"
    FoodReminder       ReminderType = "food"
    ExerciseReminder   ReminderType = "exercise"
    CustomReminder     ReminderType = "custom"
)

type Frequency string

const (
    Once    Frequency = "once"
    Daily   Frequency = "daily"
    Weekly  Frequency = "weekly"
    Monthly Frequency = "monthly"
    Custom  Frequency = "custom"
)
```

##### Chat System Models
```go
type Conversation struct {
    ID             uint              `json:"id" gorm:"primaryKey"`
    Type           ConversationType  `json:"type" gorm:"not null"`
    Title          string            `json:"title"`
    CommunityID    *uint             `json:"community_id,omitempty"`
    CreatedBy      uint              `json:"created_by" gorm:"not null"`
    IsActive       bool              `json:"is_active" gorm:"default:true"`
    AllowFileShare bool              `json:"allow_file_share" gorm:"default:true"`
    MessageRetentionDays int         `json:"message_retention_days" gorm:"default:365"`
    CreatedAt      time.Time         `json:"created_at"`
    UpdatedAt      time.Time         `json:"updated_at"`
    
    // Relationships
    Participants   []ConversationParticipant `json:"participants,omitempty"`
    Messages       []ChatMessage             `json:"messages,omitempty"`
    Community      *Community                `json:"community,omitempty" gorm:"foreignKey:CommunityID"`
}

type ConversationType string

const (
    DirectConversation    ConversationType = "direct"
    GroupConversation     ConversationType = "group"
    CommunityConversation ConversationType = "community"
)

type ConversationParticipant struct {
    ID             uint      `json:"id" gorm:"primaryKey"`
    ConversationID uint      `json:"conversation_id" gorm:"not null"`
    UserID         uint      `json:"user_id" gorm:"not null"`
    Role           ParticipantRole `json:"role" gorm:"default:'member'"`
    JoinedAt       time.Time `json:"joined_at"`
    LastReadAt     *time.Time `json:"last_read_at,omitempty"`
    IsMuted        bool      `json:"is_muted" gorm:"default:false"`
    
    // Relationships
    Conversation   Conversation `json:"conversation" gorm:"foreignKey:ConversationID"`
    User           User         `json:"user" gorm:"foreignKey:UserID"`
}

type ParticipantRole string

const (
    AdminRole     ParticipantRole = "admin"
    ModeratorRole ParticipantRole = "moderator"
    MemberRole    ParticipantRole = "member"
)

type ChatMessage struct {
    ID             uint        `json:"id" gorm:"primaryKey"`
    ConversationID uint        `json:"conversation_id" gorm:"not null"`
    SenderID       uint        `json:"sender_id" gorm:"not null"`
    MessageType    MessageType `json:"message_type" gorm:"default:'text'"`
    Content        string      `json:"content"`
    AttachmentURL  string      `json:"attachment_url,omitempty"`
    AttachmentType string      `json:"attachment_type,omitempty"`
    AttachmentSize int64       `json:"attachment_size,omitempty"`
    Mentions       []uint      `json:"mentions" gorm:"type:integer[]"`
    ReplyToID      *uint       `json:"reply_to_id,omitempty"`
    IsEdited       bool        `json:"is_edited" gorm:"default:false"`
    EditedAt       *time.Time  `json:"edited_at,omitempty"`
    IsEncrypted    bool        `json:"is_encrypted" gorm:"default:true"`
    DeliveryStatus DeliveryStatus `json:"delivery_status" gorm:"default:'sent'"`
    CreatedAt      time.Time   `json:"created_at"`
    UpdatedAt      time.Time   `json:"updated_at"`
    
    // Relationships
    Conversation   Conversation    `json:"conversation" gorm:"foreignKey:ConversationID"`
    Sender         User            `json:"sender" gorm:"foreignKey:SenderID"`
    ReplyTo        *ChatMessage    `json:"reply_to,omitempty" gorm:"foreignKey:ReplyToID"`
    Reactions      []MessageReaction `json:"reactions,omitempty"`
}

type MessageType string

const (
    TextMessage     MessageType = "text"
    ImageMessage    MessageType = "image"
    DocumentMessage MessageType = "document"
    VoiceMessage    MessageType = "voice"
    SystemMessage   MessageType = "system"
)

type DeliveryStatus string

const (
    SentStatus      DeliveryStatus = "sent"
    DeliveredStatus DeliveryStatus = "delivered"
    ReadStatus      DeliveryStatus = "read"
)

type MessageReaction struct {
    ID        uint      `json:"id" gorm:"primaryKey"`
    MessageID uint      `json:"message_id" gorm:"not null"`
    UserID    uint      `json:"user_id" gorm:"not null"`
    Emoji     string    `json:"emoji" gorm:"not null"`
    CreatedAt time.Time `json:"created_at"`
    
    // Relationships
    Message   ChatMessage `json:"message" gorm:"foreignKey:MessageID"`
    User      User        `json:"user" gorm:"foreignKey:UserID"`
}
```

##### Video Session Models
```go
type VideoSession struct {
    ID               uint            `json:"id" gorm:"primaryKey"`
    Title            string          `json:"title" gorm:"not null"`
    Description      string          `json:"description"`
    Type             SessionType     `json:"type" gorm:"not null"`
    CommunityID      *uint           `json:"community_id,omitempty"`
    HostID           uint            `json:"host_id" gorm:"not null"`
    ScheduledTime    time.Time       `json:"scheduled_time"`
    Duration         int             `json:"duration"` // minutes
    MaxParticipants  *int            `json:"max_participants,omitempty"`
    GoogleMeetLink   string          `json:"google_meet_link"`
    LinkExpiration   time.Time       `json:"link_expiration"`
    IsRecurring      bool            `json:"is_recurring" gorm:"default:false"`
    RecurringPattern string          `json:"recurring_pattern,omitempty"`
    Status           SessionStatus   `json:"status" gorm:"default:'scheduled'"`
    RequiresApproval bool            `json:"requires_approval" gorm:"default:false"`
    CreatedAt        time.Time       `json:"created_at"`
    UpdatedAt        time.Time       `json:"updated_at"`
    
    // Relationships
    Host             User                    `json:"host" gorm:"foreignKey:HostID"`
    Community        *Community              `json:"community,omitempty" gorm:"foreignKey:CommunityID"`
    Participants     []SessionParticipant    `json:"participants,omitempty"`
}

type SessionType string

const (
    GroupCommunitySession SessionType = "group_community"
    OneOnOneSession       SessionType = "one_on_one"
    ExpertSession         SessionType = "expert_session"
    WorkshopSession       SessionType = "workshop"
)

type SessionStatus string

const (
    ScheduledSession SessionStatus = "scheduled"
    ActiveSession    SessionStatus = "active"
    CompletedSession SessionStatus = "completed"
    CancelledSession SessionStatus = "cancelled"
)

type SessionParticipant struct {
    ID               uint                 `json:"id" gorm:"primaryKey"`
    SessionID        uint                 `json:"session_id" gorm:"not null"`
    UserID           uint                 `json:"user_id" gorm:"not null"`
    InvitationStatus InvitationStatus     `json:"invitation_status" gorm:"default:'invited'"`
    JoinedAt         *time.Time           `json:"joined_at,omitempty"`
    LeftAt           *time.Time           `json:"left_at,omitempty"`
    Duration         int                  `json:"duration"` // minutes participated
    CreatedAt        time.Time            `json:"created_at"`
    
    // Relationships
    Session          VideoSession         `json:"session" gorm:"foreignKey:SessionID"`
    User             User                 `json:"user" gorm:"foreignKey:UserID"`
}

type InvitationStatus string

const (
    InvitedStatus  InvitationStatus = "invited"
    AcceptedStatus InvitationStatus = "accepted"
    DeclinedStatus InvitationStatus = "declined"
    JoinedStatus   InvitationStatus = "joined"
    LeftStatus     InvitationStatus = "left"
)
```

##### Community Models
```go
type Community struct {
    ID                    uint           `json:"id" gorm:"primaryKey"`
    Name                  string         `json:"name" gorm:"not null"`
    Description           string         `json:"description"`
    Type                  CommunityType  `json:"type" gorm:"not null"`
    Category              CommunityCategory `json:"category" gorm:"not null"`
    Tags                  []string       `json:"tags" gorm:"type:text[]"`
    AdminID               uint           `json:"admin_id" gorm:"not null"`
    IsPublic              bool           `json:"is_public" gorm:"default:true"`
    RequireApproval       bool           `json:"require_approval" gorm:"default:false"`
    AllowVideoMeets       bool           `json:"allow_video_meets" gorm:"default:true"`
    AllowDirectMessaging  bool           `json:"allow_direct_messaging" gorm:"default:true"`
    ContentModerationLevel ModerationLevel `json:"content_moderation_level" gorm:"default:'moderate'"`
    MemberCount           int            `json:"member_count" gorm:"default:0"`
    IsActive              bool           `json:"is_active" gorm:"default:true"`
    CreatedAt             time.Time      `json:"created_at"`
    UpdatedAt             time.Time      `json:"updated_at"`
    
    // Relationships
    Admin                 User               `json:"admin" gorm:"foreignKey:AdminID"`
    Members               []CommunityMember  `json:"members,omitempty"`
    Posts                 []CommunityPost    `json:"posts,omitempty"`
    VideoSessions         []VideoSession     `json:"video_sessions,omitempty"`
    Conversations         []Conversation     `json:"conversations,omitempty"`
}

type CommunityType string

const (
    PublicCommunity    CommunityType = "public"
    PrivateCommunity   CommunityType = "private"
    ExpertLedCommunity CommunityType = "expert_led"
    RegionalCommunity  CommunityType = "regional"
)

type CommunityCategory string

const (
    CancerTypeCategory     CommunityCategory = "cancer_type"
    TreatmentStageCategory CommunityCategory = "treatment_stage"
    AgeGroupCategory       CommunityCategory = "age_group"
    InterestCategory       CommunityCategory = "interest"
    LocationCategory       CommunityCategory = "location"
)

type ModerationLevel string

const (
    StrictModeration   ModerationLevel = "strict"
    ModerateModeration ModerationLevel = "moderate"
    RelaxedModeration  ModerationLevel = "relaxed"
)

type CommunityMember struct {
    ID          uint        `json:"id" gorm:"primaryKey"`
    CommunityID uint        `json:"community_id" gorm:"not null"`
    UserID      uint        `json:"user_id" gorm:"not null"`
    Role        MemberRole  `json:"role" gorm:"default:'member'"`
    JoinedAt    time.Time   `json:"joined_at"`
    IsActive    bool        `json:"is_active" gorm:"default:true"`
    IsMuted     bool        `json:"is_muted" gorm:"default:false"`
    
    // Relationships
    Community   Community   `json:"community" gorm:"foreignKey:CommunityID"`
    User        User        `json:"user" gorm:"foreignKey:UserID"`
}

type MemberRole string

const (
    CommunityAdmin     MemberRole = "admin"
    CommunityModerator MemberRole = "moderator"
    CommunityMember    MemberRole = "member"
)
```

#### API Handlers Structure

##### Authentication Handler
```go
package handlers

import (
    "net/http"
    "github.com/gin-gonic/gin"
    "your-app/internal/services"
    "your-app/internal/utils"
)

type AuthHandler struct {
    authService services.AuthService
}

func NewAuthHandler(authService services.AuthService) *AuthHandler {
    return &AuthHandler{
        authService: authService,
    }
}

func (h *AuthHandler) Register(c *gin.Context) {
    var req RegisterRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    if err := utils.ValidateStruct(req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    user, err := h.authService.Register(req)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusCreated, gin.H{"user": user})
}

func (h *AuthHandler) Login(c *gin.Context) {
    var req LoginRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    token, user, err := h.authService.Login(req.Email, req.Password)
    if err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid credentials"})
        return
    }
    
    c.JSON(http.StatusOK, gin.H{
        "token": token,
        "user":  user,
    })
}
```

#### Gin Router Setup
```go
package main

import (
    "log"
    "github.com/gin-gonic/gin"
    "your-app/internal/handlers"
    "your-app/internal/middleware"
    "your-app/internal/config"
)

func setupRouter() *gin.Engine {
    r := gin.Default()
    
    // Middleware
    r.Use(middleware.CORS())
    r.Use(middleware.Logger())
    r.Use(middleware.RateLimit())
    
    // Health check
    r.GET("/health", func(c *gin.Context) {
        c.JSON(200, gin.H{"status": "ok"})
    })
    
    // API routes
    api := r.Group("/api/v1")
    {
        // Auth routes (public)
        auth := api.Group("/auth")
        {
            auth.POST("/register", authHandler.Register)
            auth.POST("/login", authHandler.Login)
            auth.POST("/forgot-password", authHandler.ForgotPassword)
            auth.POST("/reset-password", authHandler.ResetPassword)
        }
        
        // Protected routes
        protected := api.Group("/")
        protected.Use(middleware.AuthRequired())
        {
            // User management
            users := protected.Group("/users")
            {
                users.GET("/profile", userHandler.GetProfile)
                users.PUT("/profile", userHandler.UpdateProfile)
                users.DELETE("/account", userHandler.DeleteAccount)
            }
            
            // Community features
            community := protected.Group("/community")
            {
                community.GET("/posts", communityHandler.GetPosts)
                community.POST("/posts", communityHandler.CreatePost)
                community.GET("/posts/:id", communityHandler.GetPost)
                community.PUT("/posts/:id", communityHandler.UpdatePost)
                community.DELETE("/posts/:id", communityHandler.DeletePost)
                community.POST("/posts/:id/like", communityHandler.LikePost)
                community.POST("/posts/:id/comments", communityHandler.AddComment)
            }
            
            // Medical records
            medical := protected.Group("/medical")
            {
                medical.GET("/records", medicalHandler.GetRecords)
                medical.POST("/records", medicalHandler.UploadRecord)
                medical.GET("/records/:id", medicalHandler.GetRecord)
                medical.PUT("/records/:id", medicalHandler.UpdateRecord)
                medical.DELETE("/records/:id", medicalHandler.DeleteRecord)
                medical.POST("/records/:id/share", medicalHandler.ShareRecord)
            }
            
            // Reminders
            reminders := protected.Group("/reminders")
            {
                reminders.GET("/", reminderHandler.GetReminders)
                reminders.POST("/", reminderHandler.CreateReminder)
                reminders.PUT("/:id", reminderHandler.UpdateReminder)
                reminders.DELETE("/:id", reminderHandler.DeleteReminder)
                reminders.POST("/:id/mark-taken", reminderHandler.MarkTaken)
            }
            
            // AI-powered features
            ai := protected.Group("/ai")
            {
                ai.POST("/chat", aiHandler.Chat)
                ai.GET("/insights", aiHandler.GetInsights)
                ai.POST("/analyze-prescription", aiHandler.AnalyzePrescription)
                ai.GET("/diet-suggestions", aiHandler.GetDietSuggestions)
            }
            
            // Payment and subscription
            payments := protected.Group("/payments")
            {
                payments.GET("/subscription", paymentHandler.GetSubscription)
                payments.POST("/subscribe", paymentHandler.Subscribe)
                payments.POST("/cancel", paymentHandler.CancelSubscription)
                payments.GET("/billing-history", paymentHandler.GetBillingHistory)
            }
        }
    }
    
    return r
}
```

### Database Schema Overview
```sql
-- Core user tables
Users, Patients, Doctors, Caregivers, Institutions

-- Community features
CommunityPosts, Comments, Likes, UserConnections, SupportGroups

-- Medical management
MedicalRecords, Prescriptions, Appointments, Symptoms

-- Reminders and planning
Reminders, ReminderLogs, NutritionPlans, ExercisePlans

-- Analytics and insights
UserAnalytics, HealthMetrics, EngagementMetrics

-- Financial and admin
Subscriptions, Payments, AuditLogs, SystemSettings
```

### Golang Backend API Endpoints (Called from Next.js Server-Side)

**Note**: These endpoints are internal backend APIs called from Next.js API routes, not directly from the client browser.

```
GET    /health                           - Health check endpoint
POST   /api/v1/auth/register            - User registration
POST   /api/v1/auth/login               - User login
GET    /api/v1/auth/verify              - Verify JWT token
POST   /api/v1/auth/forgot-password     - Password reset request
POST   /api/v1/auth/reset-password      - Password reset confirmation
POST   /api/v1/auth/refresh             - Refresh JWT token

GET    /api/v1/users/profile            - Get user profile
PUT    /api/v1/users/profile            - Update user profile
DELETE /api/v1/users/account            - Delete user account

GET    /api/v1/community/posts          - Get community posts
POST   /api/v1/community/posts          - Create new post
GET    /api/v1/community/posts/:id      - Get specific post
PUT    /api/v1/community/posts/:id      - Update post
DELETE /api/v1/community/posts/:id      - Delete post
POST   /api/v1/community/posts/:id/like - Like/unlike post
POST   /api/v1/community/posts/:id/comments - Add comment

GET    /api/v1/chat/conversations          - Get user conversations
POST   /api/v1/chat/conversations          - Start new conversation
GET    /api/v1/chat/conversations/:id      - Get conversation messages
POST   /api/v1/chat/conversations/:id/messages - Send message
PUT    /api/v1/chat/messages/:id           - Edit message
DELETE /api/v1/chat/messages/:id          - Delete message
POST   /api/v1/chat/messages/:id/reactions - Add reaction to message

GET    /api/v1/video/sessions              - Get scheduled video sessions
POST   /api/v1/video/sessions              - Create video session
GET    /api/v1/video/sessions/:id          - Get session details
PUT    /api/v1/video/sessions/:id          - Update session
DELETE /api/v1/video/sessions/:id         - Cancel session
POST   /api/v1/video/sessions/:id/join     - Join video session
POST   /api/v1/video/sessions/:id/leave    - Leave video session
GET    /api/v1/video/sessions/:id/participants - Get session participants

GET    /api/v1/medical/records          - Get medical records
POST   /api/v1/medical/records          - Upload medical record
GET    /api/v1/medical/records/:id      - Get specific record
PUT    /api/v1/medical/records/:id      - Update record
DELETE /api/v1/medical/records/:id      - Delete record
POST   /api/v1/medical/records/:id/share - Share record

GET    /api/v1/reminders                - Get reminders
POST   /api/v1/reminders                - Create reminder
PUT    /api/v1/reminders/:id            - Update reminder
DELETE /api/v1/reminders/:id            - Delete reminder
POST   /api/v1/reminders/:id/mark-taken - Mark reminder as taken

POST   /api/v1/ai/chat                  - AI chat endpoint
GET    /api/v1/ai/insights              - Get AI insights
POST   /api/v1/ai/analyze-prescription  - Analyze prescription with AI
GET    /api/v1/ai/diet-suggestions      - Get AI diet suggestions

GET    /api/v1/payments/subscription    - Get subscription details
POST   /api/v1/payments/subscribe       - Subscribe to premium
POST   /api/v1/payments/cancel          - Cancel subscription
GET    /api/v1/payments/billing-history - Get billing history
```

### Next.js Client-Facing API Routes

These are the API routes that the Next.js frontend will call (which internally call the Golang backend):

```
POST   /api/auth/login                  - Login (calls Golang /api/v1/auth/login)
POST   /api/auth/register               - Register (calls Golang /api/v1/auth/register)
POST   /api/auth/logout                 - Logout (clears HTTP-only cookie)
GET    /api/auth/me                     - Get current user (calls Golang /api/v1/auth/verify)

GET    /api/dashboard                   - Dashboard data (batched backend calls)
GET    /api/community/posts             - Community posts (calls Golang backend)
POST   /api/community/posts             - Create post (calls Golang backend)
PUT    /api/community/posts/[id]        - Update post (calls Golang backend)
DELETE /api/community/posts/[id]        - Delete post (calls Golang backend)

GET    /api/chat/conversations          - User conversations (calls Golang backend)
POST   /api/chat/conversations          - Start conversation (calls Golang backend)
GET    /api/chat/conversations/[id]     - Conversation messages (calls Golang backend)
POST   /api/chat/conversations/[id]/messages - Send message (calls Golang backend)

GET    /api/video/sessions              - Video sessions (calls Golang backend)
POST   /api/video/sessions              - Create session (calls Golang backend)
POST   /api/video/sessions/[id]/join    - Join session (calls Golang backend)

GET    /api/medical/records             - Medical records (calls Golang backend)
POST   /api/medical/upload              - Upload record (calls Golang backend)
GET    /api/medical/records/[id]        - Get record (calls Golang backend)

GET    /api/reminders                   - User reminders (calls Golang backend)
POST   /api/reminders                   - Create reminder (calls Golang backend)
PUT    /api/reminders/[id]             - Update reminder (calls Golang backend)
POST   /api/reminders/[id]/taken       - Mark taken (calls Golang backend)

POST   /api/ai/chat                     - AI chat (calls Golang backend)
GET    /api/ai/insights                 - AI insights (calls Golang backend)

GET    /api/subscription                - Subscription info (calls Golang backend)
POST   /api/subscribe                   - Subscribe (calls Golang backend)
POST   /api/cancel-subscription         - Cancel (calls Golang backend)
```

### Golang Dependencies (go.mod)
```go
module cancer-support-app

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/golang-jwt/jwt/v5 v5.0.0
    gorm.io/gorm v1.25.0
    gorm.io/driver/postgres v1.5.0
    github.com/go-redis/redis/v8 v8.11.5
    github.com/aws/aws-sdk-go v1.44.0
    github.com/joho/godotenv v1.4.0
    github.com/swaggo/gin-swagger v1.6.0
    github.com/swaggo/swag v1.16.1
    golang.org/x/crypto v0.10.0
    github.com/google/uuid v1.3.0
    github.com/golang-migrate/migrate/v4 v4.16.0
    github.com/stretchr/testify v1.8.4
    github.com/gin-contrib/cors v1.4.0
    github.com/ulule/limiter/v3 v3.11.2
    github.com/go-playground/validator/v10 v10.14.0
    github.com/gorilla/websocket v1.5.0
)
```

### Next.js Dependencies (package.json)
```json
{
  "name": "beautiful-life-frontend",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "next": "14.0.0",
    "react": "18.2.0",
    "react-dom": "18.2.0",
    "typescript": "5.0.0",
    "@types/node": "20.0.0",
    "@types/react": "18.2.0",
    "@types/react-dom": "18.2.0",
    "tailwindcss": "3.3.0",
    "autoprefixer": "10.4.0",
    "postcss": "8.4.0",
    "axios": "1.6.0",
    "jsonwebtoken": "9.0.0",
    "@types/jsonwebtoken": "9.0.0",
    "iron-session": "8.0.0",
    "swr": "2.2.0",
    "framer-motion": "10.16.0",
    "react-hook-form": "7.47.0",
    "@hookform/resolvers": "3.3.0",
    "zod": "3.22.0",
    "date-fns": "2.30.0",
    "lucide-react": "0.290.0",
    "react-query": "3.39.0",
    "socket.io-client": "4.7.0"
  },
  "devDependencies": {
    "eslint": "8.0.0",
    "eslint-config-next": "14.0.0",
    "@typescript-eslint/eslint-plugin": "6.0.0",
    "@typescript-eslint/parser": "6.0.0",
    "prettier": "3.0.0",
    "prettier-plugin-tailwindcss": "0.5.0"
  }
}
```

---

## 🎯 Success Metrics & KPIs

### User Engagement
- Daily Active Users (DAU)
- Monthly Active Users (MAU)
- Session Duration
- Feature Adoption Rate
- Community Engagement Rate

### Health Outcomes
- Medication Adherence Rate
- Appointment Attendance Rate
- Treatment Completion Rate
- Quality of Life Scores
- User Satisfaction Scores

### Business Metrics
- Customer Acquisition Cost (CAC)
- Lifetime Value (LTV)
- Monthly Recurring Revenue (MRR)
- Churn Rate
- Net Promoter Score (NPS)

---

This business logic document serves as the foundation for developing the cancer support app. Each section can be expanded with more detailed technical specifications as development progresses.
