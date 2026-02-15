# System Design Document

## Architecture Overview

### High-Level Architecture
```
┌─────────────────────────────────────────────────┐
│         Android Mobile Application              │
│         (Optimized for Low-End Devices)         │
└──────────────┬──────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────┐
│           Application Layer                     │
│  - Camera  - Audio  - UI  - Sync Manager        │
└──────────────┬──────────────────────────────────┘
               │
       ┌───────┴────────┐
       │                │
┌──────▼─────┐   ┌─────▼──────────┐
│   Local    │   │   Cloud (AWS)  │
│   Device   │   │   (When Online)│
│            │   │                │
│ TF Lite    │   │  Amplify       │
│ SQLite     │   │  S3 Storage    │
│ Storage    │   │  Cognito       │
└────────────┘   └────────────────┘
```

### Target Users
- 150M+ rural learners across three sectors:
  - Students (K-12 education)
  - Farmers (agricultural knowledge)
  - Traders (business skills)

## Component Design

### 1. Android Mobile Application
**Technology**: Native Android (Java/Kotlin)

**Key Components**:
- Camera Module (worksheet capture)
- Audio Module (text-to-speech feedback)
- Language Selector (Tamil/Hindi)
- Sector Selector (Student/Farmer/Trader)
- AI Points Dashboard
- Reward Catalog
- Offline Status Indicator
- Sync Manager

**Optimization for Low-End Devices**:
- Minimal memory footprint
- Efficient battery usage
- Compressed assets
- Progressive loading

### 2. Edge Intelligence Layer (On-Device)

**TensorFlow Lite Models**:
- **OCR Model**: Handwriting recognition
  - Trained on Tamil and Hindi scripts
  - Optimized for worksheet formats
  - Quantized for mobile performance
  
- **NLP Model**: Language processing
  - Answer validation logic
  - Local language understanding
  - Grading algorithm

**Model Management**:
- Models bundled with app installation
- Periodic updates when online (delta updates)
- Version control and rollback capability

### 3. Local Data Layer (SQLite)

**Database Schema**:
```sql
-- Users table
Users (
  userId INTEGER PRIMARY KEY,
  username TEXT,
  sector TEXT, -- Student/Farmer/Trader
  language TEXT, -- Tamil/Hindi
  aiPoints INTEGER DEFAULT 0,
  createdAt TIMESTAMP,
  lastSyncAt TIMESTAMP
)

-- Grading History
GradingHistory (
  gradingId INTEGER PRIMARY KEY,
  userId INTEGER,
  worksheetImage BLOB,
  recognizedText TEXT,
  score INTEGER,
  audioFeedback TEXT,
  pointsEarned INTEGER,
  gradedAt TIMESTAMP,
  synced BOOLEAN DEFAULT 0,
  FOREIGN KEY (userId) REFERENCES Users(userId)
)

-- Tutorial Progress
TutorialProgress (
  progressId INTEGER PRIMARY KEY,
  userId INTEGER,
  tutorialId TEXT,
  sector TEXT,
  completed BOOLEAN DEFAULT 0,
  pointsEarned INTEGER,
  completedAt TIMESTAMP,
  synced BOOLEAN DEFAULT 0,
  FOREIGN KEY (userId) REFERENCES Users(userId)
)

-- Tutorials (Offline Cache)
Tutorials (
  tutorialId TEXT PRIMARY KEY,
  sector TEXT,
  language TEXT,
  title TEXT,
  videoPath TEXT, -- Local file path
  content TEXT,
  pointsReward INTEGER,
  downloadedAt TIMESTAMP
)

-- Rewards
Rewards (
  rewardId INTEGER PRIMARY KEY,
  userId INTEGER,
  rewardType TEXT, -- 'book_exchange', 'stationery', 'sapling'
  pointsCost INTEGER,
  status TEXT, -- 'pending', 'approved', 'delivered'
  requestedAt TIMESTAMP,
  synced BOOLEAN DEFAULT 0,
  FOREIGN KEY (userId) REFERENCES Users(userId)
)

-- Sync Queue
SyncQueue (
  queueId INTEGER PRIMARY KEY,
  dataType TEXT, -- 'grading', 'progress', 'reward'
  dataId INTEGER,
  priority INTEGER,
  createdAt TIMESTAMP
)

-- Community Impact
CommunityImpact (
  impactId INTEGER PRIMARY KEY,
  userId INTEGER,
  impactType TEXT, -- 'tree_planted', 'book_recycled', 'co2_saved'
  quantity DECIMAL,
  unit TEXT,
  recordedAt TIMESTAMP,
  synced BOOLEAN DEFAULT 0,
  FOREIGN KEY (userId) REFERENCES Users(userId)
)

-- Success Stories
SuccessStories (
  storyId INTEGER PRIMARY KEY,
  userId INTEGER,
  sector TEXT,
  storyText TEXT,
  imageData BLOB,
  likes INTEGER DEFAULT 0,
  sharedAt TIMESTAMP,
  synced BOOLEAN DEFAULT 0,
  FOREIGN KEY (userId) REFERENCES Users(userId)
)

-- Referrals
Referrals (
  referralId INTEGER PRIMARY KEY,
  referrerId INTEGER,
  referredUserId INTEGER,
  bonusPoints INTEGER,
  status TEXT, -- 'pending', 'completed'
  createdAt TIMESTAMP,
  FOREIGN KEY (referrerId) REFERENCES Users(userId),
  FOREIGN KEY (referredUserId) REFERENCES Users(userId)
)

-- Analytics (Local)
UserAnalytics (
  analyticsId INTEGER PRIMARY KEY,
  userId INTEGER,
  weakTopics TEXT, -- JSON array
  recommendedTutorials TEXT, -- JSON array
  learningStreak INTEGER,
  lastUpdated TIMESTAMP,
  FOREIGN KEY (userId) REFERENCES Users(userId)
)
```

### 4. Cloud Services (AWS)

**AWS Amplify**:
- **Authentication**: AWS Cognito for user management
- **API**: GraphQL/REST endpoints for data sync
- **DataStore**: Conflict resolution for offline-first sync
- **Storage**: User profile and progress backup

**Amazon S3**:
- Video tutorial library
- Model updates repository
- User-generated content backup

**Sync Strategy**:
- Background sync when WiFi available
- Prioritized sync queue (rewards > progress > history)
- Conflict resolution (server wins for points, merge for progress)
- Retry mechanism with exponential backoff

## Data Flow

### VoiceAI Grader Flow (Offline)
```
1. User captures worksheet photo
   ↓
2. Image preprocessed (crop, enhance, normalize)
   ↓
3. TensorFlow Lite OCR model processes image
   ↓
4. Recognized text extracted
   ↓
5. NLP model validates answers
   ↓
6. Score calculated + AI Points awarded
   ↓
7. Text-to-Speech generates audio feedback (Tamil/Hindi)
   ↓
8. Results saved to SQLite (GradingHistory table)
   ↓
9. Entry added to SyncQueue for later upload
   ↓
10. User hears audio feedback and sees score
```

### Offline-to-Sync Loop
```
┌─────────────────────────────────────────────┐
│         OFFLINE MODE (No Internet)          │
│                                             │
│  User Activity → Local Processing          │
│       ↓                                     │
│  TensorFlow Lite (OCR/NLP)                 │
│       ↓                                     │
│  SQLite Storage                            │
│       ↓                                     │
│  SyncQueue (Pending Items)                 │
│                                             │
└──────────────┬──────────────────────────────┘
               │
               │ Internet Connection Detected
               ↓
┌──────────────────────────────────────────────┐
│         SYNC MODE (Online)                   │
│                                              │
│  1. Check SyncQueue for pending items        │
│  2. Prioritize: Rewards > Progress > History │
│  3. Upload to AWS Amplify API                │
│  4. Receive server confirmation              │
│  5. Update synced flag in SQLite             │
│  6. Download new tutorials (if available)    │
│  7. Download model updates (if available)    │
│  8. Clear SyncQueue                          │
│                                              │
└──────────────┬───────────────────────────────┘
               │
               │ Sync Complete
               ↓
         Return to Offline Mode
```

### Tutorial Access Flow
```
ONLINE (First Time):
User selects tutorial → Check local cache → Not found
  → Download from S3 → Save to SQLite → Play video

OFFLINE (Subsequent):
User selects tutorial → Check local cache → Found
  → Load from SQLite → Play video → Track progress
  → Award points → Add to SyncQueue
```

### Reward Redemption Flow
```
1. User browses reward catalog (offline)
   ↓
2. User selects reward (e.g., "Exchange old books for stationery")
   ↓
3. System checks AI Points balance (local SQLite)
   ↓
4. If sufficient points:
   - Deduct points locally
   - Create reward request in SQLite
   - Add to SyncQueue (HIGH PRIORITY)
   ↓
5. When online:
   - Sync reward request to AWS
   - Admin approves/processes
   - Status updated on next sync
```

## Language Support Implementation

### Approach
- Language packs stored locally in app bundle
- UI strings in JSON format
- TTS (Text-to-Speech) engines for Tamil and Hindi
- OCR models trained on both scripts

### Structure
```
/assets
  /locales
    /ta.json  (Tamil)
    /hi.json  (Hindi)
  /tts
    /tamil_voice.flite
    /hindi_voice.flite
  /models
    /ocr_tamil.tflite
    /ocr_hindi.tflite
    /nlp_combined.tflite
```

### Text-to-Speech Implementation
- Use Android TTS API with offline voices
- Fallback to bundled Flite voices if system TTS unavailable
- Adjustable speech rate for accessibility

## Security Design

### Privacy-First Architecture
- All data processed locally before any cloud transmission
- User consent required for sync operations
- No sensitive data (worksheet images, personal info) transmitted without explicit permission

### Authentication
- AWS Cognito for user management
- JWT tokens for API access
- Offline authentication using cached credentials
- Biometric authentication option (fingerprint)

### Data Protection
- Encrypted SQLite database (SQLCipher)
- Secure local file storage
- HTTPS for all network calls
- No PII in logs or analytics

## Deployment Strategy

### Phase 1: MVP (Pilot - 1,000 users)
- Single state deployment (Tamil Nadu or Bihar)
- Student sector only
- Basic VoiceAI grader
- Limited tutorial library (10 modules)
- Simple reward catalog

### Phase 2: Expansion (100,000 users)
- Multi-state deployment
- All three sectors (Students, Farmers, Traders)
- Enhanced grading accuracy
- Expanded tutorial library (50+ modules)
- Physical reward fulfillment network

### Phase 3: Scale (10M+ users)
- National deployment
- Additional languages (Telugu, Bengali, etc.)
- Advanced AI features (personalized learning paths)
- Community features (peer learning)
- Partnership with NGOs for reward distribution

### Phase 4: Impact (150M+ users)
- Pan-India coverage
- Full feature set
- Sustainability model (micro-transactions, sponsorships)
- Impact measurement and reporting
- Partnership with government schemes (Digital India, Skill India)
- Integration with existing education/agriculture programs
- Research publication on digital divide reduction
- Open-source components for broader ecosystem impact

## Technology Decisions

### Why AWS Amplify?
- Rapid development with offline-first architecture
- Built-in conflict resolution for sync
- Seamless AWS integration
- Auto-scaling for 150M+ users
- Cost-effective for social impact projects

### Why TensorFlow Lite?
- Optimized for mobile and low-end devices
- Cross-platform support (Android)
- Small model size (<20MB per model)
- Fast inference (<5 seconds)
- Offline-first AI capabilities

### Why Native Android?
- Better performance on low-end devices
- Smaller app size vs cross-platform frameworks
- Direct access to device features (camera, TTS)
- Lower battery consumption
- Wider device compatibility (Android 6.0+)

### Why SQLite?
- Zero-configuration embedded database
- Reliable offline storage
- ACID compliance for data integrity
- Efficient for mobile devices
- Encryption support (SQLCipher)

## Impact Measurement Framework

### Digital Divide Metrics
- Number of rural learners reached
- Geographic coverage (villages/districts)
- Device accessibility (low-end smartphone penetration)
- Offline usage percentage (target: >80%)

### Learning Outcomes
- Pre/post assessment score improvement
- Tutorial completion rates by sector
- Time to competency reduction
- Skill acquisition tracking

### Environmental Impact
- Trees planted through sapling rewards
- CO2 offset from digital learning (vs physical tutoring travel)
- Books recycled through exchange program
- Waste reduction metrics

### Economic Impact
- Cost savings for learners (vs paid tutoring)
- Income improvement for farmers (better practices)
- Business growth for traders (new skills)
- Job creation through reward fulfillment network

### Social Impact
- Community engagement levels
- Peer-to-peer knowledge sharing
- Success story propagation
- Referral network growth
- Digital literacy improvement

## Performance Considerations

### Optimization Strategies for Low-End Devices
- Model quantization (8-bit integer models)
- Lazy loading of tutorials
- Image compression before OCR
- Background sync (WiFi only by default)
- Efficient SQLite queries with indexing
- Memory management (clear cache regularly)
- Battery optimization (batch operations)

### Monitoring
- App performance metrics (crash rate, ANR)
- OCR accuracy rate
- Grading time (target: <5 seconds)
- Sync success rate
- User engagement (daily active users)
- Tutorial completion rate
- Reward redemption rate
- Offline usage percentage
- Social impact metrics (trees planted, CO2 saved, books recycled)
- Accessibility usage (screen reader, voice input adoption)
- Community engagement (peer help, story shares, referrals)
- Learning outcomes (improvement over time)
