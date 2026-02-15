# Project Requirements

## Project Overview
AWS AI application for Bharath with offline capabilities, local language support, and gamification features.

## Core Objective
Bridge the digital divide for 150M+ rural learners by providing tools that work without internet.

## Core Requirements (EARS Notation)

### 1. VoiceAI Grader
- When a user snaps a photo of a worksheet, the system shall read the handwriting using OCR
- When handwriting is recognized, the system shall analyze the answers for correctness
- When grading is complete, the system shall provide instant audio feedback in Tamil or Hindi
- When audio grading is requested, the system shall use text-to-speech in the user's selected language

### 2. Offline Mode
- When the device has no internet connection, the system shall allow all core grading features to function
- When the device has no internet connection, the system shall allow all tutorial features to function
- When offline data is generated, the system shall store it locally in SQLite
- When internet connection is restored, the system shall automatically sync stored data to the cloud
- When syncing occurs, the system shall preserve user progress and reward points

### 3. Reward Ecosystem (AI Points)
- When a user completes a grading task, the system shall award AI Points
- When a user completes a tutorial module, the system shall award AI Points
- When a user accumulates points, the system shall display available physical rewards
- When a user requests reward redemption, the system shall allow exchange of old books for new stationery
- When a user requests reward redemption, the system shall allow exchange of points for saplings/plants (for farmers)
- When a reward is redeemed, the system shall update the user's point balance

### 4. Multi-Sector Tutorials
- When a student user logs in, the system shall provide student-focused learning modules
- When a farmer user logs in, the system shall provide agriculture-focused learning modules
- When a trader user logs in, the system shall provide business/trade-focused learning modules
- When tutorials are accessed offline, the system shall load content from local storage
- When new tutorials are available online, the system shall download them for offline use

### 5. Local Language Support
- When a user selects Tamil, the system shall display all UI elements in Tamil
- When a user selects Hindi, the system shall display all UI elements in Hindi
- When audio feedback is generated, the system shall use the user's selected language
- When tutorials are displayed, the system shall show content in the user's selected language

### 6. Accessibility & Inclusion
- When a visually impaired user opens the app, the system shall provide full screen reader support
- When a user has low literacy, the system shall provide icon-based navigation with audio labels
- When a user speaks their answer, the system shall accept voice input as an alternative to written worksheets
- When content is displayed, the system shall support adjustable font sizes and high-contrast modes

### 7. Community & Social Impact
- When a user completes 10 gradings, the system shall unlock peer-to-peer help features
- When a farmer shares a success story, the system shall allow other farmers to view it offline
- When a user refers another learner, the system shall award bonus AI Points to both users
- When community milestones are reached (e.g., 1000 trees planted), the system shall celebrate with all users

### 8. Progress Analytics & Insights
- When a student completes assessments, the system shall identify weak topics and recommend targeted tutorials
- When a farmer tracks crop cycles, the system shall provide seasonal learning recommendations
- When progress data syncs, the system shall generate monthly learning reports for users
- When patterns indicate struggle, the system shall adjust difficulty and provide encouragement

### 9. Sustainability & Environmental Impact
- When users earn points through learning, the system shall track carbon offset from digital learning (vs travel to tutoring centers)
- When saplings are redeemed, the system shall track trees planted and CO2 impact
- When old books are exchanged, the system shall track waste reduction metrics
- When the app syncs, the system shall display community environmental impact dashboard

## Technical Stack

### Frontend
- **Android**: Mobile application optimized for low-end smartphones
- **Native Android UI**: Lightweight interface for resource-constrained devices

### Edge Intelligence (On-Device AI)
- **TensorFlow Lite**: On-device machine learning
  - OCR for handwriting recognition
  - NLP for local language processing (Tamil/Hindi)
  - Offline inference capabilities
  - Optimized for low-end devices

### Local Data Layer
- **SQLite**: Local database for offline storage
  - Progress tracking
  - Tutorial content storage
  - Reward points tracking
  - Pending sync queue

### Cloud Services (AWS)
- **AWS Amplify**: Periodic data synchronization
  - Authentication
  - API management
  - Offline-to-cloud sync
- **Amazon S3**: Video tutorial library hosting
- **AWS Cognito**: User authentication

## Functional Requirements

### User Features
- User registration and authentication (syncs when online)
- Profile management with user sector selection (Student/Farmer/Trader)
- Language preference selection (Tamil/Hindi)
- AI Points dashboard showing current balance
- Reward catalog with redemption options
- Offline mode indicator

### VoiceAI Grader Features
- Camera integration for worksheet capture
- Handwriting OCR processing
- Answer validation and grading logic
- Text-to-speech audio feedback
- Grading history storage

### Tutorial Features
- Sector-specific content delivery
- Video playback (offline-capable)
- Progress tracking per module
- Completion rewards
- Adaptive learning paths based on performance
- Peer success stories (cached offline)

### Accessibility Features
- Screen reader compatibility (TalkBack support)
- Voice input for answers
- Icon-based navigation for low-literacy users
- Adjustable font sizes and high-contrast mode
- Audio-first interface option

### Community Features
- Peer-to-peer help (unlocked after 10 gradings)
- Success story sharing (offline-viewable)
- Referral system with bonus points
- Community milestone celebrations
- Local leaderboards (village/district level)

### Reward System Features
- AI Points accumulation tracking
- Physical reward catalog (books, stationery, saplings)
- Redemption request management
- Transaction history
- Environmental impact tracking (trees planted, CO2 offset, waste reduced)
- Community impact dashboard
- Referral bonuses

## Non-Functional Requirements

### Performance
- Grading response time: <5 seconds on low-end devices
- App launch time: <3 seconds
- OCR accuracy: >85% for handwritten text
- Audio feedback latency: <2 seconds
- Efficient model loading (<50MB total)
- Minimal battery consumption (<5% per hour of active use)

### Scalability
- Support 150M+ users at scale
- Handle 10M+ concurrent offline users
- Efficient sync for millions of daily transactions
- Handle various Android device types (Android 6.0+)
- Expandable language support (future: Telugu, Bengali, Marathi)

### Accessibility
- WCAG 2.1 Level AA compliance target
- Screen reader compatible
- Voice navigation support
- Works for users with limited digital literacy
- Supports users with visual impairments

### Reliability
- 99.9% offline functionality uptime
- Graceful degradation when storage is low
- Data integrity during sync failures
- Automatic retry mechanisms
- No data loss during app crashes

### Social Impact Metrics
- Track number of learners reached
- Measure learning outcomes improvement
- Calculate environmental impact (trees planted, CO2 saved)
- Monitor digital divide reduction
- Track community engagement levels

### Security
- All data must be processed locally to ensure user privacy before syncing
- Encrypted local SQLite database
- Secure user credentials storage
- Privacy-compliant data handling
- No sensitive data transmitted without user consent

## Constraints
- Must work on low-end Android smartphones
- Limited device storage (optimize model and tutorial sizes)
- Intermittent or no network connectivity in rural areas
- Low bandwidth when connectivity is available
- Battery efficiency critical for extended offline use
- Users may have limited digital literacy
