BridgePass — Bridging the JHS to SHS Gap in GhanaBridgePass is an offline-first, adaptive learning web application engineered to address critical foundational learning deficits among Junior High School (JHS) graduates in Ghana as they transition into Senior High School (SHS). Built to align directly with the Ghana Education Service (GES) national curriculum, BridgePass combines diagnostic assessment algorithms, localized micro-lessons in native Ghanaian languages, and role-based monitoring dashboards to ensure academic readiness regardless of internet connectivity or geographical constraints.   Use Case DiagramCode snippetflowchart TD
    classDef student fill:#EEF2FF,stroke:#4F46E5,stroke-width:2px,color:#1E1B4B;
    classDef parent fill:#FEF3C7,stroke:#F59E0B,stroke-width:2px,color:#78350F;
    classDef teacher fill:#D1FAE5,stroke:#10B981,stroke-width:2px,color:#064E3B;
    classDef system fill:#F1F5F9,stroke:#64748B,stroke-width:1px,color:#0F172A;

    subgraph Actors
        S[👨‍🎓 Student]:::student
        P[👨‍👩‍👧 Parent / Guardian]:::parent
        T[🏫 Teacher]:::teacher
    end

    subgraph "BridgePass Core System"
        UC1(Take 5-Question Placement Diagnostic):::system
        UC2(Receive Adaptive Bridge Plan):::system
        UC3(Study Offline Micro-Lessons):::system
        UC4(Switch Content Language EN/Twi/Ga/Ewe):::system
        UC5(Complete Practice Quiz & Earn Credits):::system
        UC6(Generate 6-Character Invite Code):::system
        
        UC7(Enter Student Invite Code):::system
        UC8(View Child Readiness & Weak Areas):::system
        
        UC9(Monitor Class Readiness Metrics):::system
        UC10(Identify At-Risk Students & Common Gaps):::system
    end

    S --> UC1
    S --> UC2
    S --> UC3
    S --> UC4
    S --> UC5
    S --> UC6

    P --> UC7
    P --> UC8

    T --> UC7
    T --> UC9
    T --> UC10
Tech StackFrontend Framework: Vanilla HTML5, CSS3, ES6+ JavaScript (Zero external framework dependencies)   Design & Styling: CSS Custom Properties (Variables), Flexbox, CSS Grid, Glassmorphism backdrop-blur effects, Responsive Mobile-First Architecture   Persistence & Offline Engine: HTML5 localStorage for complete offline state management, local profile creation, progress tracking, and cached curriculum delivery   Target Backend Infrastructure: Supabase (PostgreSQL, Row Level Security, Realtime Database Subscriptions, GoTrue Auth)Visualization & UI Components: Native DOM manipulation, custom state event-listeners, CSS animation engine   Project StructurePlaintextbridgepass/
├── index.html                   # Single-Page Application containing markup, styles, runtime logic, and curriculum data
├── README.md                    # Project documentation
└── supabase/
    └── migrations/
        └── 20261008000000_init_bridgepass.sql  # Database schema, foreign keys, and RLS policies
Database SchemaSQL-- Enable UUID Extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 1. Profiles Table (Users)
CREATE TABLE profiles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE NOT NULL,
    full_name TEXT NOT NULL,
    role TEXT NOT NULL CHECK (role IN ('student', 'parent', 'teacher')),
    grade_level TEXT DEFAULT 'JHS 3 Graduate',
    avatar_char VARCHAR(2) NOT NULL,
    readiness_score INT DEFAULT 0 CHECK (readiness_score BETWEEN 0 AND 100),
    credits INT DEFAULT 0,
    streak_days INT DEFAULT 0,
    invite_code VARCHAR(6) UNIQUE NOT NULL,
    school_name TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 2. Lessons Table
CREATE TABLE lessons (
    id TEXT PRIMARY KEY, -- e.g., 'm1', 'm2', 'e1', 's1'
    subject_key TEXT NOT NULL CHECK (subject_key IN ('math', 'english', 'science')),
    title TEXT NOT NULL,
    grade_level TEXT NOT NULL,
    duration_label TEXT NOT NULL,
    difficulty INT DEFAULT 1 CHECK (difficulty BETWEEN 1 AND 5),
    is_gap BOOLEAN DEFAULT FALSE,
    content_json JSONB NOT NULL, -- Stores translations (english, twi, ga, ewe)
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 3. Student Progress Table
CREATE TABLE student_progress (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    student_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
    lesson_id TEXT NOT NULL REFERENCES lessons(id) ON DELETE CASCADE,
    completed BOOLEAN DEFAULT FALSE,
    progress_pct INT DEFAULT 0 CHECK (progress_pct BETWEEN 0 AND 100),
    last_score INT DEFAULT 0,
    completed_at TIMESTAMP WITH TIME ZONE,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(student_id, lesson_id)
);

-- 4. Guardian Links Table
CREATE TABLE guardian_links (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    guardian_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
    student_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'approved', 'rejected')),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(guardian_id, student_id)
);

-- 5. Badges Master Table
CREATE TABLE badges (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    icon_symbol TEXT NOT NULL,
    description TEXT NOT NULL,
    credits_reward INT DEFAULT 0
);

-- 6. User Badges Junction Table
CREATE TABLE user_badges (
    user_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
    badge_id TEXT NOT NULL REFERENCES badges(id) ON DELETE CASCADE,
    earned_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    PRIMARY KEY (user_id, badge_id)
);
Database RelationshipsCode snippeterDiagram
    PROFILES ||--o{ STUDENT_PROGRESS : "tracks completion"
    LESSONS ||--o{ STUDENT_PROGRESS : "referenced in"
    PROFILES ||--o{ GUARDIAN_LINKS : "acts as guardian"
    PROFILES ||--o{ GUARDIAN_LINKS : "acts as student"
    PROFILES ||--o{ USER_BADGES : "unlocks"
    BADGES ||--o{ USER_BADGES : "awarded via"

    PROFILES {
        uuid id PK
        string email UK
        string full_name
        string role
        string invite_code UK
        int readiness_score
        int credits
        int streak_days
    }

    LESSONS {
        string id PK
        string subject_key
        string title
        boolean is_gap
        jsonb content_json
    }

    STUDENT_PROGRESS {
        uuid id PK
        uuid student_id FK
        string lesson_id FK
        boolean completed
        int progress_pct
    }

    GUARDIAN_LINKS {
        uuid id PK
        uuid guardian_id FK
        uuid student_id FK
        string status
    }

    BADGES {
        string id PK
        string name
        string icon_symbol
        int credits_reward
    }

    USER_BADGES {
        uuid user_id PK,FK
        string badge_id PK,FK
        timestamp earned_at
    }
Key Features & RationaleOffline-First ArchitectureEducational & Business Rationale: High data costs and unstable broadband access across rural and peri-urban Ghana present a critical barrier to continuous learning. Requiring constant online access exacerbates educational inequality.   Technical Implementation: Core assets, UI logic, and structured JSON curriculum data (covering Mathematics, English, and Science) are embedded directly within the application footprint. User interactions, progress metrics, and offline session states persist locally via HTML5 localStorage and queue background synchronization upon detecting active network connectivity.   Adaptive DiagnosticEducational & Business Rationale: Broad, un-targeted revision fails to resolve specific foundational misunderstandings stemming from lower primary grades.   Technical Implementation: Upon initialization, students complete a 5-question diagnostic quiz evaluating core competencies across Primary 5 through JHS 3 concepts. The diagnostic mapping engine pinpoint gaps—such as Primary 5 division or JHS 1 fractions—to instantly tailor the user's study sequence.   Bridge PlanEducational & Business Rationale: JHS graduates require an explicit, structured remediation pathway prior to starting SHS 1 to prevent immediate academic falling-behind.   Technical Implementation: The system constructs an individualized "Bridge Plan" dashboard view. Identified knowledge gaps are dynamically tagged with higher priority (isGap: true) and positioned at the top of the student's lesson pipeline.   Credit & Streak SystemEducational & Business Rationale: Self-directed study requires strong intrinsic and extrinsic behavioral motivators to maintain habituation.Technical Implementation: Completing micro-lessons, maintaining consecutive active days, and achieving high scores on practice assessments awards "Transition Credits" (e.g., +10 credits per practice set) and increments active daily streak counters. Unlocked achievements grant system badges stored within user state.   Guardian LinkingEducational & Business Rationale: Parental oversight and teacher monitoring significantly improve student accountability and learning outcomes.Technical Implementation: Each student profile generates an algorithmic, 6-character unique alphanumeric invite code (e.g., KW4M2P). Parents or teachers input this code within their respective dashboard views to establish a linked relationship. Linked accounts receive real-time visibility into student readiness percentages, weak-area alerts, and completed lesson logs.   Localised ContentEducational & Business Rationale: STEM subjects in Ghana are taught exclusively in English, creating a dual cognitive burden for students still acquiring English proficiency. Explaining complex concepts in native languages enhances conceptual comprehension.   Technical Implementation: The platform features single-tap localization switching between English, Twi, Ga, and Ewe. Every micro-lesson contains parallel linguistic structures for explanations, worked step-by-step examples, and practice question feedback.   DevelopmentPrerequisitesAny standard modern web browser (Google Chrome, Mozilla Firefox, Apple Safari, Microsoft Edge).A local static HTTP server (Optional, e.g., Python http.server, Node.js serve, or Live Server extension).Quick Start GuideClone the Repository:Bashgit clone https://github.com/your-username/bridgepass.git
cd bridgepass
Launch Application Locally:Option A (Direct File Browser Opening): Double-click index.html to open directly in your web browser.   Option B (Local Web Server):Bashpython3 -m http.server 8000
Access the application at http://localhost:8000.Sample Pre-Configured Test AccountsThe platform includes built-in mock accounts accessible directly via the Sign In modal interface:   RoleEmail AddressPasswordMock Data Profile FeaturesStudentstudent@bridgepass.com   student123   Kwame Asante (72% Readiness, 7-day streak, 340 credits, 12 completed lessons)   Parentparent@bridgepass.com   parent123   Mrs. Akua Asante (Linked to 2 children, weak-area monitoring)   Teacherteacher@bridgepass.com   teacher123   Mr. Emmanuel Osei (34 Students, Class analytics, At-risk tracking)   Supabase MigrationsTo transition BridgePass from localStorage client-side state to a multi-tenant cloud persistence engine using Supabase, apply the following SQL configuration script:SQL-- 1. Enable Row Level Security
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE lessons ENABLE ROW LEVEL SECURITY;
ALTER TABLE student_progress ENABLE ROW LEVEL SECURITY;
ALTER TABLE guardian_links ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_badges ENABLE ROW LEVEL SECURITY;

-- 2. Profiles RLS Policies
CREATE POLICY "Public profiles viewable by authenticated users" 
    ON profiles FOR SELECT 
    TO authenticated 
    USING (true);

CREATE POLICY "Users can update their own profile" 
    ON profiles FOR UPDATE 
    TO authenticated 
    USING (auth.uid() = id);

-- 3. Student Progress RLS Policies
CREATE POLICY "Students can read own progress" 
    ON student_progress FOR SELECT 
    TO authenticated 
    USING (auth.uid() = student_id);

CREATE POLICY "Students can upsert own progress" 
    ON student_progress FOR ALL 
    TO authenticated 
    USING (auth.uid() = student_id);

CREATE POLICY "Guardians can view linked student progress" 
    ON student_progress FOR SELECT 
    TO authenticated 
    USING (
        EXISTS (
            SELECT 1 FROM guardian_links
            WHERE guardian_links.guardian_id = auth.uid()
              AND guardian_links.student_id = student_progress.student_id
              AND guardian_links.status = 'approved'
        )
    );

-- 4. Guardian Links RLS Policies
CREATE POLICY "Users can view links involving themselves" 
    ON guardian_links FOR SELECT 
    TO authenticated 
    USING (auth.uid() = guardian_id OR auth.uid() = student_id);

CREATE POLICY "Guardians can request linking" 
    ON guardian_links FOR INSERT 
    TO authenticated 
    WITH CHECK (auth.uid() = guardian_id);

CREATE POLICY "Students can approve or reject links" 
    ON guardian_links FOR UPDATE 
    TO authenticated 
    USING (auth.uid() = student_id);

-- 5. Trigger Functions for Automatic Timestamp Updates
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_profiles_timestamp
    BEFORE UPDATE ON profiles
    FOR EACH ROW EXECUTE FUNCTION update_timestamp();

CREATE TRIGGER update_student_progress_timestamp
    BEFORE UPDATE ON student_progress
    FOR EACH ROW EXECUTE FUNCTION update_timestamp();
Academic MetadataStudent / Author: B. Arhin Forson   Index Number: 6396424   Project Supervisor: Dr. E.O. Oppong   Institution & Region: Academic Capstone Research Project, Accra, Ghana 🇬🇭[cite: 1]
