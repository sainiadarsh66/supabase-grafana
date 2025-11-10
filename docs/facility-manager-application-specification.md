# **Facility Manager - Complete Application Specification**

## **📋 Overview**

**"Facility Manager - The Old Rectory"** is a full-stack task management web application designed specifically for hospitality operations. It streamlines the workflow for capturing, triaging, assigning, and resolving housekeeping and maintenance issues in a facility with multiple buildings and rooms.

**Target Environment:** Tablet-optimized (can be used on desktop/mobile)  
**Primary Use Case:** Hotel/hospitality facility operations management  
**Key Value Proposition:** Visual issue reporting with photo capture, real-time task tracking, role-based dashboards, and automated task routing

---

## **🏗️ Technology Stack**

### **Frontend**
- **React 18** with TypeScript
- **Vite** for build tooling
- **React Router v6** for navigation
- **TanStack Query (React Query)** for data fetching/caching
- **Tailwind CSS** for styling
- **shadcn/ui** component library (Radix UI primitives)
- **Framer Motion** for animations
- **date-fns** for date manipulation
- **Lucide React** for icons
- **Zod** + **React Hook Form** for form validation

### **Backend (Lovable Cloud / Supabase)**
- **Supabase** (PostgreSQL database)
- **Supabase Auth** for authentication
- **Supabase Storage** for photo uploads
- **Edge Functions** (Deno) for serverless logic
- **Real-time subscriptions** for live updates
- **Row Level Security (RLS)** for data access control

### **UI/UX Features**
- Particles.js animated backgrounds
- Aurora background effects
- Dark/Light theme support
- Fullscreen mode
- Audio alerts for high-priority tasks
- Loading skeletons
- Toast notifications
- Responsive design (mobile/tablet/desktop)

---

## **👥 User Roles & Permissions**

### **1. Owner (Admin)**
- Full system access
- View all tasks across departments
- Manage users and assign roles
- Access calendar view, board view, and all dashboards
- Create, edit, delete, and assign tasks
- Configure recurring tasks
- View activity logs
- Access settings and user management

### **2. Maintenance**
- View all maintenance-category tasks
- Update task status (new → assigned → in_progress → completed)
- Assign tasks to themselves
- Upload "after" photos when completing tasks
- Add notes to tasks
- Report new maintenance issues
- Enable audio alerts for high-priority tasks

### **3. Housekeeping**
- View all housekeeping-category tasks
- Organized by teams: Upstairs, Downstairs, Cedar, Laundry
- Update task status
- Assign tasks to themselves
- Upload completion photos
- Report new housekeeping issues
- Enable audio alerts

### **4. Staff (Reporter)**
- Can only report issues (photo or text)
- No access to task management dashboards
- Minimal permissions

---

## **🗄️ Database Schema**

### **Tables**

#### **`profiles`**
```sql
- id (uuid, FK to auth.users)
- email (text)
- full_name (text)
- housekeeping_team (text) -- upstairs|downstairs|cedar|laundry
- created_at (timestamp)
- updated_at (timestamp)
```

#### **`user_roles`**
```sql
- id (uuid)
- user_id (uuid, FK to profiles)
- role (enum: owner|maintenance|housekeeping|staff)
- created_at (timestamp)
```

#### **`tasks`**
```sql
- id (uuid)
- title (text, required)
- description (text, nullable)
- categories (text[], e.g., ['housekeeping'] or ['maintenance'])
- status (text: new|assigned|in_progress|completed)
- priority (text: low|medium|high)
- location_building (text, nullable)
- location_room (text, nullable)
- photos (text[], URLs of "before" photos)
- after_photos (text[], URLs of "after" photos)
- assignee (text, email)
- assigned_users (uuid[], array of user IDs)
- housekeeping_teams (text[], e.g., ['upstairs', 'laundry'])
- created_by (uuid, FK to profiles)
- assigned_by (uuid, FK to profiles)
- created_via (text: photo|text)
- scheduled_date (timestamp, nullable)
- original_scheduled_date (timestamp, nullable -- for carried tasks)
- started_at (timestamp, nullable)
- completed_at (timestamp, nullable)
- notes (text, nullable)
- created_at (timestamp)
- updated_at (timestamp)

-- Recurring task fields:
- recurrence_type (text: none|daily|weekly|monthly)
- recurrence_interval (integer, default 1)
- recurrence_days_of_week (integer[], 0=Sun, 6=Sat)
- recurrence_end_date (timestamp, nullable)
- is_recurring_parent (boolean, default false)
- parent_task_id (uuid, FK to tasks, nullable)
```

#### **`activity_logs`**
```sql
- id (uuid)
- user_id (uuid, FK to profiles)
- action (text: created|updated|deleted|assigned)
- entity_type (text: task|user)
- entity_id (uuid)
- details (jsonb, structured log data)
- created_at (timestamp)
```

### **Database Functions**

1. **`handle_new_user()`** - Trigger on auth.users INSERT
   - Creates profile entry
   - Assigns "owner" role to first user

2. **`update_updated_at_column()`** - Trigger for updated_at timestamps

3. **`has_role(_user_id, _role)`** - Check if user has specific role

4. **`log_task_change()`** - Trigger on tasks INSERT/UPDATE
   - Logs task creation and status changes to activity_logs

5. **`delete_recurring_task_instances()`** - Trigger on tasks DELETE
   - Cascades deletion of recurring task child instances

### **Storage Buckets**

- **`task-photos`** (private bucket)
  - Stores before/after photos
  - Organized by user_id or task_id
  - Uses signed URLs (1-hour expiration)

---

## **🔐 Authentication & Security**

### **Authentication**
- Email/password authentication via Supabase Auth
- Persistent sessions with localStorage
- Auto-refresh tokens
- Protected routes with role-based access control
- Automatic redirection to `/auth` for unauthenticated users

### **Row Level Security (RLS) Policies**

#### **tasks table:**
- **SELECT**: Owners see all; Maintenance sees maintenance tasks; Housekeeping sees housekeeping tasks
- **INSERT**: Users can create tasks as themselves
- **UPDATE**: Users can update tasks in their domain or tasks they created
- **DELETE**: Only owners can delete tasks

#### **profiles table:**
- **SELECT**: Users see their own profile; Owners see all
- **UPDATE**: Users can update their own profile
- **INSERT/DELETE**: Blocked (handled by trigger)

#### **user_roles table:**
- **SELECT**: Users see their own roles; Owners see all
- **INSERT/DELETE**: Only owners
- **UPDATE**: Blocked

#### **activity_logs table:**
- **SELECT**: Only owners
- **INSERT**: Service role only (via trigger)
- **UPDATE/DELETE**: Owners only; no updates allowed

---

## **🌐 Application Routes**

```tsx
/auth                    - Login/Signup page (public)
/                        - Staff Portal (protected)
/board                   - Owner's Task Board (owner only)
/maintenance             - Maintenance Dashboard (maintenance role)
/housekeeping            - Housekeeping Dashboard (housekeeping role)
/task/:id                - Task Detail Page (protected)
/calendar                - Calendar View (owner only)
/recurring-tasks         - Recurring Tasks Management (owner only)
/report/photo            - Report Issue with Photo (protected)
/report/text             - Report Issue with Text (protected)
/settings                - Settings Hub (owner only)
/settings/users          - User Management (owner only)
/activity-logs           - Activity Logs (owner only)
/*                       - 404 Not Found
```

---

## **📱 Core Features & Pages**

### **1. Authentication Page (`/auth`)**
- Tabbed interface: Sign In | Sign Up
- Email + Password fields
- Full name field (sign up only)
- Animated particle background (theme-aware)
- Auto-redirect on successful auth
- First user becomes "owner" automatically

---

### **2. Staff Portal (`/` - Index.tsx)**

**For Non-Owner Users:**
- Welcome message with user's full name
- Large action buttons:
  - 📷 Report with Photo
  - 📝 Report in Text
- "What's your role?" section explaining permissions
- Logout button
- Animated particles background

**For Owner Users:**
- Full dashboard with task management
- Date navigation (previous/next day, today button)
- Progress bar showing completion rate
- Daily statistics cards:
  - Total tasks
  - Completed tasks
  - In-progress tasks
  - High-priority count
- Category filter tabs: All | Maintenance | Housekeeping
- Status filter tabs: All | New | In Progress | Completed
- Kanban-style columns: New | In Progress | Completed
- Task cards with:
  - Title, priority badge, location
  - Photo indicators
  - Assigned user info
  - Quick actions (view details, start, complete)
- Quick access buttons:
  - 🔊 Audio alerts toggle
  - ⛶ Fullscreen toggle
  - ➕ Create Task
  - 📅 Calendar View
  - ⚙️ Settings
- Real-time updates via Supabase subscriptions
- Particles background effect

---

### **3. Maintenance Dashboard (`/maintenance`)**

**Designed for maintenance staff workflow:**
- Date navigation with completion progress bar
- Filter tabs: All | New | Assigned | In Progress | Completed
- Task counter badges on each tab
- Grid of task cards showing:
  - Priority badge (color-coded)
  - Title and description
  - Location (room/area)
  - Status icon
  - Assigned user
  - Created timestamp
  - "Carried forward" badge (if rescheduled)
  - Timer showing time since creation/start
- Card actions:
  - Click to view full details
  - Assign to me
  - Update status
  - View photos
- Header buttons:
  - 🔊 Audio alerts (enables continuous alarm for high-priority tasks)
  - ⛶ Fullscreen toggle
  - 📷 Report Issue dropdown (Photo/Text)
  - 🏠 Home button
  - Logout
- Empty state with "Report New Issue" button
- Real-time task updates
- Audio alerts for high-priority tasks (continuous until started)
- Auto-stop alerts when task status changes to in_progress/completed

---

### **4. Housekeeping Dashboard (`/housekeeping`)**

**Team-based task organization:**
- Date navigation with progress bar
- Statistics: Total, Completed, In Progress, High Priority
- Filter tabs by status (All/New/Assigned/In Progress/Completed)
- **Four Team Sections:**
  - 🏠 Upstairs Team
  - 🏢 Downstairs Team
  - 🌲 Cedar Team
  - 🧺 Laundry Team
- Each team shows filtered task cards
- Task cards include:
  - Priority and status
  - Location
  - Assign/Status actions
  - Photos indicator
  - Carried-forward badge
- Header actions:
  - Fullscreen toggle
  - Report Issue dropdown
  - Audio alerts toggle
  - Logout
- Sort dropdown: Priority | Status | Date
- Real-time updates per team
- Audio alerts for high-priority tasks

---

### **5. Task Board (`/board`)**

**Owner's comprehensive task view:**
- Filter tabs: All Tasks | Today | Urgent | My Tasks | Overdue
- Date range picker with quick filters:
  - Today
  - This Week
  - This Month
  - Custom range (calendar picker)
- List view of all tasks with:
  - Status icon
  - Title
  - Photo/After-photo counts
  - Category badges
  - Location
  - Priority
  - Scheduled date
  - Assignee
  - Created timestamp
- Click task card to view details
- Real-time updates

---

### **6. Calendar View (`/calendar`)**

**Visual scheduling interface:**
- Month navigation (previous/next, today button)
- Full calendar grid with:
  - Day numbers
  - Current day highlight
  - Selected day highlight
  - Task count badges per day
  - Priority dots (color-coded)
- **Filter Controls:**
  - Category tabs: All | Maintenance | Housekeeping | Other
  - Priority dropdown
  - Status dropdown
  - "Recurring Only" toggle button
- Sidebar showing tasks for selected date
- Unscheduled tasks section at bottom
- Task actions:
  - Click to view details
  - Schedule (drag or click)
  - Delete
- "Add Task" button (creates task with prefilled date)
- Create Task Dialog with:
  - Title, Description
  - Category, Priority
  - Location (building/room)
  - Scheduled date
  - Assign users (multi-select)
  - Housekeeping teams (multi-select if housekeeping)
- Real-time calendar updates

---

### **7. Task Detail Page (`/task/:id`)**

**Full task management interface:**
- Back button (routes to appropriate dashboard based on role)
- Task header:
  - Title (large)
  - Status badge
  - Priority badge
  - More actions menu (Duplicate | Delete | Make Recurring)
- **Before Photos Section:**
  - Photo gallery grid
  - Click to view full-size
- **Editable Description** (inline editing for owners)
- **Details Section:**
  - 🏢 Building
  - 📍 Room
  - 📅 Scheduled Date
  - 👤 Created By
  - 🕐 Created At
  - ⏱️ Started At (when status → in_progress)
  - ✅ Completed At (when status → completed)
- **Recurrence Info** (if recurring):
  - Pattern description (e.g., "Every 2 weeks on Mon, Wed")
  - End date (if set)
- **Status Update Section:**
  - Radio buttons: New | Assigned | In Progress | Completed
  - Save Status button
- **Completion Section** (when status = completed):
  - "After" photos upload
  - Preview uploaded photos
  - Auto-attach to task on status update
- **Notes Section:**
  - Textarea for work notes
  - Save Notes button
- **Delete Confirmation:**
  - If recurring instance: Option to delete single or all instances
  - Shows instance count
  - Cascades to parent task if deleting all
- Real-time updates via subscription

---

### **8. Report Issue - Photo (`/report/photo`)**

**Visual issue reporting:**
- Photo capture interface:
  - Camera input (mobile devices use camera directly)
  - Multiple photo support
  - Photo preview grid
  - Remove photo button (X)
- Form fields:
  - Issue Title *
  - Department * (Housekeeping | Maintenance)
  - Room/Area *
  - Priority * (Low | Medium | High)
  - Description (optional)
- **Recurring Task Toggle:**
  - Make this recurring checkbox
  - Configure Recurrence button
- Recurring Dialog:
  - Type: Daily | Weekly | Monthly
  - Interval: Every X day(s)/week(s)/month(s)
  - Days of week (if weekly)
  - End date (optional)
- Submit/Cancel buttons
- Uploads photos to Supabase Storage
- Creates task via edge function
- Generates recurring instances immediately if configured

---

### **9. Report Issue - Text (`/report/text`)**

**Text-based issue reporting:**
- Form fields:
  - Department *
  - Room/Area *
  - Priority *
  - Issue Title *
  - Detailed Description * (textarea)
- Recurring Task Toggle (same as photo report)
- Submit/Cancel buttons
- Creates task via edge function
- Generates recurring instances if configured

---

### **10. Recurring Tasks Management (`/recurring-tasks`)**

**Manage all recurring task patterns:**
- List of all recurring parent tasks
- Each card shows:
  - Title, Description
  - Priority badge
  - Category badges
  - Location
  - Recurrence pattern description
  - Start date
  - Created date
- Actions per task:
  - ✏️ Edit (opens pattern editor)
  - 🗑️ Delete (with confirmation)
- **Edit Pattern Dialog:**
  - Recurrence Type dropdown
  - Interval input
  - Days of week checkboxes (if weekly)
  - End date picker
- Delete confirmation warns about instance deletion
- Real-time updates via subscription

---

### **11. Settings (`/settings`)**

**Settings hub (owner only):**
- Cards for:
  - Locations (placeholder - not implemented)
  - Users & PINs → navigates to `/settings/users`
  - Activity Logs → navigates to `/activity-logs`
  - Checklists (placeholder)
  - Notifications (placeholder)

---

### **12. Users Management (`/settings/users`)**

**User administration (owner only):**
- Table showing all users:
  - Email
  - Full Name
  - Roles (badges)
  - Team (if housekeeping)
  - Actions dropdown
- **Actions per user:**
  - Assign Role (opens dialog)
  - Assign to Team (if housekeeping role):
    - Upstairs | Downstairs | Cedar | Laundry
  - Remove Team
  - Remove specific role
- **Assign Role Dialog:**
  - Shows available roles (not already assigned)
  - Role descriptions:
    - Owner: Full system access
    - Maintenance: Access maintenance tasks
    - Housekeeping: Access housekeeping tasks
    - Staff: Report issues only
- Real-time updates via subscription

---

### **13. Activity Logs (`/activity-logs`)**

**Audit trail (owner only):**
- (Detailed implementation not shown in provided files, but table structure exists)
- Shows task creation, updates, assignments
- User actions with timestamps
- JSONB details field for structured logging

---

## **⚙️ Edge Functions (Backend Logic)**

### **1. `create-task`**
**Purpose:** Create new tasks (bypasses RLS for initial creation)

**Input:**
```json
{
  "category": "housekeeping" | "maintenance",
  "title": "string",
  "description": "string",
  "room": "string",
  "priority": "low" | "medium" | "high",
  "createdBy": "uuid"
}
```

**Logic:**
- Validates input data (required fields, string lengths)
- Authenticates user via JWT
- Ensures createdBy matches authenticated user
- Uses service role to insert task (bypasses RLS)
- Returns task ID and data

**Output:**
```json
{
  "task": { "id": "uuid", ... }
}
```

---

### **2. `auto-assign-housekeeping`**
**Purpose:** Automatically assign housekeeping tasks to team members

**Input:**
```json
{
  "taskId": "uuid"
}
```

**Logic:**
- Fetches task details
- Identifies housekeeping team(s) from task
- Queries team members (users with housekeeping role + matching team)
- Calculates workload for each team member (active assigned tasks)
- Assigns to member with lowest workload
- Updates task status to 'assigned'

**Output:**
```json
{
  "success": true,
  "assignedTo": "user@email.com"
}
```

---

### **3. `process-pending-housekeeping`**
**Purpose:** Cron job to auto-assign pending housekeeping tasks

**Trigger:** Scheduled (e.g., every 30 minutes)

**Logic:**
- Authenticates via CRON_SECRET
- Fetches 'new' housekeeping tasks older than 20 minutes
- Calls `auto-assign-housekeeping` for each task
- Logs results (processed, assigned, skipped, errors)

**Output:**
```json
{
  "processed": 5,
  "assigned": 4,
  "skipped": 1,
  "errors": []
}
```

---

### **4. `carry-forward-tasks`**
**Purpose:** Cron job to move incomplete tasks to next day

**Trigger:** Scheduled (e.g., nightly at midnight)

**Logic:**
- Authenticates via CRON_SECRET
- Queries incomplete tasks (new, assigned, in_progress)
- Filters non-recurring tasks scheduled for today or before
- Updates `scheduled_date` to tomorrow
- Sets `original_scheduled_date` if not already set
- Returns count of carried tasks

**Output:**
```json
{
  "processed": 12,
  "message": "Carried forward 12 tasks"
}
```

---

### **5. `generate-recurring-tasks`**
**Purpose:** Generate future instances of recurring tasks

**Input:**
```json
{
  "taskId": "uuid",
  "daysAhead": 90
}
```

**Logic:**
- Fetches recurring parent task
- Validates recurrence configuration
- Calculates dates based on:
  - Daily: Every N days
  - Weekly: Every N weeks on specified days
  - Monthly: Every N months on same day
- Stops at recurrence_end_date or daysAhead limit
- Skips dates that already have instances
- Creates child tasks with:
  - Same title, description, location, priority, categories
  - `parent_task_id` set to parent
  - `is_recurring_parent = false`
  - `scheduled_date` set to calculated date
- Returns count of created tasks

**Output:**
```json
{
  "tasksCreated": 42,
  "message": "Generated 42 recurring task instances"
}
```

---

## **🔔 Real-time Features**

### **Supabase Realtime Subscriptions:**

Each page subscribes to relevant table changes:

```tsx
const channel = supabase
  .channel('channel-name')
  .on('postgres_changes', {
    event: '*',  // INSERT, UPDATE, DELETE
    schema: 'public',
    table: 'tasks',
    filter: 'category=eq.maintenance'  // optional
  }, (payload) => {
    // Refetch data or update local state
  })
  .subscribe();
```

**Subscribed Events:**
- Tasks table: All pages refetch on any change
- User roles table: Users Management refetches on change
- Activity logs table: Activity Logs refetches on insert

---

## **🔊 Audio Alert System**

### **Custom Hook: `useAudioAlert()`**

**Purpose:** Manage continuous audio alerts for high-priority tasks

**Features:**
- Plays continuous alarm sound (looping audio)
- Tracks multiple active alerts (by task ID)
- Browser permission request (user interaction required)
- Auto-initialization on first use
- Alert lifecycle:
  1. Start alert when high-priority task appears
  2. Continue looping until stopped
  3. Stop when task status changes to in_progress/completed

**Methods:**
```tsx
{
  initializeAudio: () => Promise
  startAlert: (taskId: string) => Promise
  stopAlert: (taskId: string) => void
  stopAllAlerts: () => void
  getActiveAlerts: () => string[]
  isInitialized: boolean
}
```

**Usage:**
- Maintenance & Housekeeping dashboards monitor tasks
- Show visual prompt to enable audio
- Button in header to manually enable
- Toast notifications when alerts start/stop

---

## **📊 Key UX/UI Patterns**

### **1. Loading States**
- Skeleton loaders for Maintenance/Housekeeping
- Spinner for quick loads
- Skeleton cards match actual content layout

### **2. Empty States**
- Friendly messages: "All clear! No tasks yet"
- Call-to-action buttons: "Report New Issue"
- Context-aware messaging based on filters

### **3. Toast Notifications**
- Success: ✅ Green toast
- Error: ❌ Red toast (destructive variant)
- Info: Default toast
- Auto-dismiss after 3-10 seconds
- Positioned top-right on desktop, top-center on mobile

### **4. Confirmation Dialogs**
- AlertDialog for destructive actions (delete task)
- Explains consequences
- Cancel / Confirm buttons

### **5. Task Cards**
- Color-coded priority badges:
  - 🔴 High (red/destructive)
  - 🟡 Medium (yellow/warning)
  - 🟢 Low (gray/secondary)
- Status icons:
  - ⏱️ New/Assigned (warning yellow)
  - 🕐 In Progress (blue)
  - ✅ Completed (green)
- Hover effects for interactivity
- Click to expand/navigate

### **6. Date Navigation**
- Previous/Next buttons (< / >)
- Current date display (e.g., "Sunday, November 10, 2025")
- "Today" quick button
- Smooth transitions

### **7. Progress Indicators**
- Progress bar showing % complete
- Color transitions: red → yellow → green
- Daily completion statistics

### **8. Responsive Design**
- Desktop: Multi-column layouts
- Tablet: Optimized for touch (primary target)
- Mobile: Stacked layouts, larger touch targets
- Fullscreen mode for kiosk/tablet installations

---

## **🎨 Theme System**

### **next-themes Integration**
- System, Light, Dark modes
- Persisted in localStorage
- CSS variables for colors
- Theme toggle in settings (not shown but available via shadcn)

### **Particle Backgrounds**
- Theme-aware colors:
  - Light mode: Black particles (#000000)
  - Dark mode: White particles (#ffffff)
- Smooth, subtle animations
- Mouse-following magnetism effect
- Performance-optimized with canvas

---

## **🔄 Recurring Tasks System**

### **Concepts:**

1. **Recurring Parent Task:**
   - `is_recurring_parent = true`
   - Stores recurrence pattern
   - Not displayed in active task lists
   - Editable in Recurring Tasks Management page

2. **Recurring Instance (Child Task):**
   - `parent_task_id` points to parent
   - `is_recurring_parent = false`
   - Functions as normal task
   - Can be completed/edited independently
   - Deleting parent cascades to all children

3. **Recurrence Types:**
   - **Daily:** Every N days
   - **Weekly:** Every N weeks on selected days (Sun-Sat)
   - **Monthly:** Every N months on same day

4. **Instance Generation:**
   - Triggered on parent creation/update
   - Edge function `generate-recurring-tasks`
   - Generates instances up to 90 days ahead
   - Respects `recurrence_end_date`
   - Scheduled job regenerates weekly

5. **Carried Tasks:**
   - When incomplete task is carried forward, `original_scheduled_date` is preserved
   - Badge shown: "🔄 Carried"
   - Helps track overdue/rescheduled work

---

## **📸 Photo Management**

### **Upload Flow:**
1. User captures/selects photo(s)
2. Frontend creates File objects
3. Preview generated via `URL.createObjectURL()`
4. On submit:
   - Upload to Supabase Storage (`task-photos` bucket)
   - Generate signed URL (1-hour expiration)
   - Store URL in task `photos` or `after_photos` array
5. Display photos in task detail with gallery view

### **Security:**
- Private bucket (not publicly accessible)
- Signed URLs with expiration
- RLS policies control bucket access
- Photos organized by user_id or task_id

---

## **🧪 Testing Considerations** (for recreation)

When recreating, consider testing:

1. **Role-based Access:**
   - Users can only access permitted routes
   - RLS policies enforce data visibility
   - First user becomes owner

2. **Real-time Updates:**
   - Multiple users see changes instantly
   - Task updates propagate across dashboards
   - Subscription cleanup on unmount

3. **Audio Alerts:**
   - Browser permission handling
   - Alert start/stop lifecycle
   - Multiple concurrent alerts

4. **Recurring Tasks:**
   - Instance generation accuracy
   - Timezone handling
   - Cascade deletion

5. **Photo Upload:**
   - Large file handling
   - Multiple simultaneous uploads
   - Signed URL expiration

6. **Date/Time Handling:**
   - Timezone consistency (UTC stored, local display)
   - Date navigation accuracy
   - Carried task date logic

7. **Edge Cases:**
   - Empty states
   - Network failures
   - Concurrent edits
   - Deleted tasks still in view

---

## **🚀 Deployment Setup**

1. **Environment Variables:**
```env
VITE_SUPABASE_URL=
VITE_SUPABASE_PUBLISHABLE_KEY=
VITE_SUPABASE_PROJECT_ID=
```

2. **Supabase Configuration:**
   - Enable Email Auth
   - Configure Site URL and Redirect URLs
   - Set up Storage bucket: `task-photos`
   - Deploy Edge Functions
   - Schedule cron jobs:
     - `process-pending-housekeeping` (every 30 min)
     - `carry-forward-tasks` (daily at midnight)
     - `generate-recurring-tasks` (weekly)

3. **Secrets (Edge Functions):**
   - `CRON_SECRET` for scheduled jobs
   - `SUPABASE_URL`
   - `SUPABASE_SERVICE_ROLE_KEY`
   - `OPENAI_API_KEY` (if using AI features)

4. **Database Setup:**
   - Run migrations in order
   - Enable RLS on all tables
   - Apply RLS policies
   - Create database functions and triggers

---

## **📝 Migration Path** (to recreate)

1. **Initialize Project:**
   - Create Vite + React + TypeScript project
   - Install dependencies (see package.json)
   - Set up Tailwind CSS + shadcn/ui

2. **Authentication:**
   - Create Supabase project
   - Set up Auth pages
   - Implement ProtectedRoute wrapper
   - Configure role checking

3. **Database Schema:**
   - Create tables (profiles, user_roles, tasks, activity_logs)
   - Add RLS policies
   - Create functions and triggers

4. **Core Pages:**
   - Build pages in order of dependency:
     1. Auth
     2. Index (staff portal + owner dashboard)
     3. Report pages (photo/text)
     4. Maintenance dashboard
     5. Housekeeping dashboard
     6. Task detail
     7. Calendar view
     8. Board view
     9. Settings pages

5. **Edge Functions:**
   - Deploy `create-task`
   - Deploy `auto-assign-housekeeping`
   - Deploy `generate-recurring-tasks`
   - Set up cron jobs

6. **Features:**
   - Add real-time subscriptions
   - Implement audio alerts
   - Add particles/aurora backgrounds
   - Configure recurring tasks
   - Set up photo upload

7. **Polish:**
   - Loading states
   - Error handling
   - Toast notifications
   - Responsive design
   - Theme support

---

## **🎯 Key Design Decisions**

1. **Tablet-First:** UI scaled for touch interaction (120-150% zoom on desktop)
2. **Visual Reporting:** Photo capture prioritized for quick issue logging
3. **Role-Based Dashboards:** Each role sees only relevant information
4. **Real-time Updates:** No manual refresh needed
5. **Audio Alerts:** Ensures high-priority tasks get immediate attention
6. **Automated Assignment:** Reduces owner workload
7. **Recurring Tasks:** Handles predictable maintenance cycles
8. **Carried Tasks:** Tracks incomplete work across days

---

This specification should provide everything needed to recreate the application from scratch. The key is following the database schema, RLS policies, and role-based permissions structure while building out the UI components progressively.
