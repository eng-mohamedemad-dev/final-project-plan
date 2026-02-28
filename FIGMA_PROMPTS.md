# Figma Design Prompts — Google Stitch

> هذا الملف يحتوي على 3 بروابت تفصيلية لتصميم الـ Figma باستخدام أداة Google Stitch.
> كل بروبت مستقل — انسخه وضعه في Google Stitch لتحصل على تصميم شامل.

---

## Prompt 1: Officer Flutter App (تطبيق الضابط)

```
Design a complete mobile app UI for a "CrimeLens" — a Crime Prediction & Response Officer App built with Flutter. This app is used by police officers in the field who receive AI-detected crime alerts and respond to incidents. The design must be modern, professional, Arabic-first (RTL) with English support, and optimized for quick one-handed use in urgent situations.

### Brand & Visual Identity
- App name: "CrimeLens — ضابط"
- Primary color: Deep Navy Blue (#1B2A4A)
- Accent color: Amber/Gold (#F59E0B) for alerts and actions
- Danger/Critical color: Red (#EF4444)
- Success color: Emerald Green (#10B981)
- Background: Clean white (#FAFBFC) with subtle gray cards
- Typography: Bold headlines for urgency, clean body text
- Dark mode supported
- Use a shield/badge icon motif for the officer theme
- Status bar indicators with color coding throughout

### Screens to Design (10 screens total):

---

#### Screen 1: Splash Screen
- App logo (shield with AI eye icon) centered
- App name "CrimeLens" below logo
- Subtle loading animation (pulsing shield or scanning line)
- Navy blue gradient background (#1B2A4A → #2D3F6B)

---

#### Screen 2: Login Screen
- Clean centered login form
- App logo at top (smaller)
- Email input field with email icon
- Password input field with eye toggle visibility icon
- "تسجيل الدخول" (Login) button — full width, amber/gold color
- "نسيت كلمة المرور؟" (Forgot password?) link below
- NO language toggle (language is controlled by admin from the dashboard, fetched from server on startup)
- Subtle background pattern (abstract city/security theme)
- Form validation: show red error text below each field if invalid
- Loading state: button shows spinner when submitting

---

#### Screen 3: Home / Crime List Screen (Main Screen)
- **Top bar**: 
  - Officer name + rank on left
  - Status indicator dot (green=available, yellow=busy, red=offline) — tappable to change status
  - Notification bell icon with red badge count on right
  - Profile avatar circle on far right (tappable → profile)
- **Status banner** below top bar: 
  - Shows current status: "متاح" (Available) in green, "مشغول" (Busy) in yellow, "غير متصل" (Offline) in red
  - Quick tap to change status with bottom sheet selector
- **Filter tabs** below: "الكل" (All), "مُكلف" (Assigned to me), "قريبة" (Nearby <5km)
- **Crime Cards List** (scrollable, pull-to-refresh):
  Each card shows:
  - Severity strip on left edge (green=low, yellow=medium, orange=high, red=critical)
  - Crime type icon + name in Arabic (e.g. "🔪 سرقة بالإكراه")
  - Status badge: color-coded pill (pending=gray, assigned=blue, in_progress=orange, visited=green)
  - Camera name + location (e.g. "كاميرا المدخل الرئيسي — محطة القاهرة")
  - Time ago (e.g. "منذ 5 دقائق")
  - Distance (e.g. "2.3 كم")
  - Confidence score percentage with small progress bar
  - If assigned to this officer: highlighted border + "مُكلف لك" badge
- **Floating action button (FAB)**: Map icon — opens map view of nearby crimes
- **Bottom navigation bar**: Home (active), Map, Chat, Notifications, Profile
- Include empty state design: illustration + "لا توجد جرائم حالياً" when no crimes

---

#### Screen 4: Crime Detail Screen
- **Hero section**: 
  - Full-width map showing crime location pin (red pulsing marker)
  - Camera location pin (blue)
  - Officer's current location pin (green)
  - Distance line between officer and crime
  - Map takes top 40% of screen
- **Crime info card** (below map, scrollable):
  - Crime type with icon + severity badge (critical=red pulsing)
  - Status: large colored pill
  - Crime date/time
  - Camera name + location
  - Confidence score: circular progress indicator with percentage
  - Assigned officer info (if assigned)
- **Evidence section**:
  - Video player thumbnail (play button overlay) — tappable to play crime scene video
  - Scene time range: "من 14:28 إلى 14:32"
- **Action buttons section** (CONTEXTUAL based on crime status):
  - If status = "assigned" (assigned to this officer):
    - Large green button: "قبول المهمة" (Accept Mission) with checkmark icon
    - Secondary red button: "رفض" (Decline) with X icon
  - If status = "in_progress":
    - Large blue button: "وصلت للموقع" (I Arrived) with location pin icon
  - If status = "visited":
    - Large green button: "تم الحل" (Resolved) with checkmark
    - Text area: optional notes field
  - If declining: 
    - Bottom sheet with text area for "سبب عدم الزيارة" (No-visit reason) — required field
    - Submit button
- **🚨 Alarm Button** (ALWAYS visible at bottom-right):
  - Large circular red pulsing button with alarm/siren icon
  - Label: "تشغيل الإنذار" (Trigger Alarm)
  - Tapping shows confirmation: "هل تريد تشغيل إنذار الكاميرا؟"
  - When alarm is active: button changes to "إيقاف الإنذار" (Stop Alarm) — animated pulsing red
  - This triggers the camera alarm for the crime's associated camera

---

#### Screen 5: Map View Screen
- **Full-screen Google Map**
- Color-coded crime markers:
  - Red pulsing = critical severity
  - Orange = high
  - Yellow = medium
  - Green = low
  - Gray = resolved
- Officer's current location: blue dot with accuracy circle
- Bottom sheet (draggable): list of nearby crimes sorted by distance
- Each item in bottom sheet: crime type, severity color, distance, tap to navigate to detail
- Filter chips at top of map: filter by severity, status
- "الجرائم القريبة خلال 5 كم" (Nearby crimes within 5km) label

---

#### Screen 6: Chat Screen (with Police Station)
- **Header**: Police station name + avatar + online indicator
- **Message list** (WhatsApp-style):
  - Sent messages: right-aligned, amber/gold background, white text
  - Received messages: left-aligned, light gray background, dark text
  - Each message shows: text, timestamp, read indicator (double checkmark)
  - Date separators between different days
- **Input area** at bottom:
  - Text input field with placeholder "اكتب رسالة..." (Type a message...)
  - Send button (arrow icon) — appears only when text is typed
- Show typing indicator when other party is typing
- Empty state: "ابدأ محادثة مع مركز الشرطة" (Start a conversation with the police station)

---

#### Screen 7: Notifications Screen
- **Header**: "الإشعارات" (Notifications) with "تحديد الكل كمقروء" (Mark all as read) action
- **Notification list**:
  - Unread notifications: white background with blue left border
  - Read notifications: slightly grayed
  - Each notification shows:
    - Icon (based on type: 🚨 crime, 📍 assignment, 💬 message, ⚠️ alert)
    - Title in bold
    - Description text
    - Time ago
  - Notification types:
    - "🚨 تم رصد جريمة جديدة" (New crime detected) — red icon
    - "📍 تم تكليفك بمهمة" (You've been assigned) — blue icon
    - "⚠️ تصعيد — لم يتم الاستجابة" (Escalation — no response) — orange icon
    - "💬 رسالة جديدة من المركز" (New message from station) — green icon
- Tap notification → navigate to relevant screen
- Pull-to-refresh
- Empty state: "لا توجد إشعارات" with bell illustration

---

#### Screen 8: Profile Screen
- **Profile header**:
  - Large circular avatar with camera/edit icon overlay
  - Officer name (large, bold)
  - Rank (e.g. "ملازم أول")
  - Badge number (e.g. "B-1234")
  - National ID
  - Police station name
- **Editable fields section** (form):
  - Name
  - Email
  - Phone
  - Change password (current + new + confirm) — expandable section
- **Status section**:
  - Current status with toggle radio buttons: متاح / مشغول / غير متصل
  - On-shift indicator (read-only, set by station)
- **Info section** (read-only):
  - Last location update time
  - Today's stats: crimes handled, response time avg
- **Actions**:
  - "حفظ التعديلات" (Save Changes) button
  - "تسجيل الخروج" (Logout) button — red, at bottom
- Note: NO language switcher — language is server-controlled by admin

---

#### Screen 9: Status Toggle Bottom Sheet
- Slide-up bottom sheet (triggered from home screen status tap)
- Three options with radio buttons:
  - 🟢 "متاح" (Available) — "ستتلقى مهام جديدة" (You'll receive new assignments)
  - 🟡 "مشغول" (Busy) — "لن تتلقى مهام جديدة" (Won't receive new assignments)
  - 🔴 "غير متصل" (Offline) — "غير متاح للخدمة" (Not available for service)
- "تأكيد" (Confirm) button
- Current status pre-selected

---

#### Screen 10: Crime Alert Overlay / Push Notification
- Design an urgent full-screen overlay that appears when a critical crime is assigned:
  - Red pulsing border animation
  - 🚨 Large siren icon at top
  - "مهمة عاجلة!" (Urgent Mission!) in large bold text
  - Crime type + severity
  - Camera name + location
  - Distance from officer
  - Confidence score
  - Two large buttons: "قبول" (Accept — green) and "التفاصيل" (Details — blue)
  - Auto-dismiss after 30 seconds if no action
  - Sound/vibration pattern
```

---

## Prompt 2: Police Station Flutter App (تطبيق مركز الشرطة)

```
Design a complete mobile app UI for "CrimeLens" — a Crime Prediction & Management Police Station App built with Flutter. This app is used by police station managers to manage officers, cameras, monitor crimes in real-time, and communicate with officers. The design must be modern, professional, Arabic-first (RTL) with English support, feature-rich with data visualization.

### Brand & Visual Identity
- App name: "CrimeLens — مركز الشرطة"
- Primary color: Deep Navy Blue (#1B2A4A)
- Secondary color: Slate Blue (#3B5998)
- Accent color: Amber/Gold (#F59E0B) 
- Background: Clean white (#FAFBFC) with card-based layout
- Dark mode supported
- Use a command center / headquarters icon motif
- Professional dashboard aesthetic with charts and metrics
- Consistent card-based UI with shadows and rounded corners

### Screens to Design (19 screens total):

---

#### Screen 1: Splash Screen
- Same brand as officer app but with "مركز الشرطة" (Police Station) subtitle
- Shield logo with command center icon variation
- Navy blue gradient background

---

#### Screen 2: Login Screen
- Same structure as officer app login
- Different subtitle: "لوحة إدارة مركز الشرطة" (Police Station Management Panel)
- Full width login form with email + password
- Login button in amber/gold
- NO language toggle (language is server-controlled by admin)

---

#### Screen 3: Dashboard / Home Screen
- **Top bar**: Station name, notification bell with badge, profile avatar
- **Stats cards row** (horizontal scroll or 2x2 grid):
  - Total Officers (icon: people) with available/total count (e.g. "12/20 متاح")
  - Total Cameras (icon: camera) with active/total (e.g. "13/15 نشطة")
  - Crimes Today (icon: alert) with number
  - Pending Crimes (icon: clock) with number — highlighted if > 0
- **Crimes by Status** section:
  - Horizontal bar chart or donut chart showing: pending, assigned, in_progress, visited, resolved, escalated, false_alarm
  - Each segment color-coded
  - Tappable to filter crime list
- **Recent Crimes** section:
  - List of 5 most recent crimes with: type, severity color strip, status badge, time ago, camera name
  - "عرض الكل" (View All) link → crimes list
- **Quick Actions** section:
  - "إضافة ضابط" (Add Officer) button
  - "إضافة كاميرا" (Add Camera) button
  - "عرض الخريطة" (View Map) button
- **Bottom Navigation**: Dashboard (active), Officers, Cameras, Crimes, More (→ Chat, Profile, Settings)

---

#### Screen 4: Officers List Screen
- **Search bar** at top: "البحث بالاسم أو البريد..." (Search by name or email)
- **Filter chips**: الكل (All), متاح (Available), مشغول (Busy), غير متصل (Offline)
- **Officer cards list** (each card):
  - Avatar circle (with colored border matching status: green/yellow/red)
  - Name (bold)
  - Rank (subtitle, gray)
  - Badge number
  - Status pill: color-coded (green=available, yellow=busy, red=offline)
  - On-shift indicator (checkmark or X)
  - Last location update: "آخر تحديث: منذ 3 دقائق"
  - Swipe actions: Edit (blue), Delete (red)
  - Tap → officer detail/edit
- **FAB**: "+" button → create officer screen
- **Action bar button**: Import icon (Excel upload)
- Empty state: illustration + "لا يوجد ضباط — أضف ضابطك الأول"

---

#### Screen 5: Officer Create/Edit Form
- Back arrow + title: "إضافة ضابط جديد" or "تعديل بيانات الضابط"
- Form fields (scrollable):
  - Avatar upload: large circle with camera icon, tap to pick from gallery
  - Name (required): text input
  - Email (required): email input with validation
  - Phone (required): phone input with country code +20
  - Password: password input (only for create, optional on edit)
  - National ID (الرقم القومي): 14-digit number input
  - Rank (الرتبة): dropdown (ملازم, ملازم أول, نقيب, رائد, مقدم, عقيد, عميد, لواء)
  - Badge Number (رقم الشارة): text input
  - On-Shift toggle switch: "في الخدمة" (On Duty)
- **Save button**: "حفظ" — full width, amber/gold
- **Reset Password button** (edit mode only): "إعادة تعيين كلمة المرور" — secondary button
  - Opens bottom sheet with new password field + confirm
- Form validation with inline red error messages

---

#### Screen 6: Officers Excel Import Screen
- **Instructions area**: 
  - "رفع ملف Excel لإضافة ضباط بشكل جماعي" (Upload Excel file to bulk add officers)
  - Show expected columns: الاسم، البريد، الهاتف، كلمة المرور، الرقم القومي، الرتبة، رقم الشارة
  - "تحميل القالب" (Download Template) link
- **Upload area**: 
  - Large dashed border drop zone with upload icon
  - "اختر ملف Excel أو اسحبه هنا" (Choose Excel file or drag here)
  - File picker button
  - Accepted formats: .xlsx, .xls, .csv
- **Progress state**: Upload progress bar with percentage
- **Results state** (after upload):
  - Success count: "✅ تم إضافة 18 ضابط بنجاح"
  - Error count: "❌ 2 صف بها أخطاء"
  - Error detail list: row number + error message (e.g. "صف 5: البريد مستخدم بالفعل")
  - "عرض الضباط" (View Officers) button

---

#### Screen 7: Officers Map Screen (Real-time Location Tracking)
- **Full-screen Google Map**
- Officer markers on map:
  - Green markers: available officers (with name label)
  - Yellow markers: busy officers
  - Red/Gray markers: offline officers
  - Each marker shows officer avatar miniature
- Tap marker → bottom sheet with officer info:
  - Name, rank, badge number
  - Status
  - Last update time
  - "اتصال" (Call) and "محادثة" (Chat) quick actions
- **Legend** at bottom: color explanation
- **Toggle**: Show/hide offline officers
- Real-time: officers move on map as they send GPS updates

---

#### Screen 8: Cameras List Screen
- **Search bar**: "البحث بالاسم أو الموقع..."
- **Filter chips**: الكل, نشطة (Active), غير نشطة (Inactive)
- **Camera cards list** (each card):
  - Camera icon with status indicator (green dot = active, red dot = inactive)
  - Camera name (bold)
  - Location name (subtitle)
  - IP address (monospace font, gray)
  - Storage type badge: "بطاقة SD" / "سحابي" / "بدون تخزين"
  - Active/Inactive toggle switch
  - Coordinates: lat, lng (small, gray)
  - Swipe actions: Edit (blue), Delete (red), Control (green — opens camera control)
- **FAB**: "+" → create camera
- **Action bar**: Import (Excel upload) icon
- **Camera Control icon** on each card → camera control screen
- **Live View icon** (eye icon) on each active camera card → opens live RTSP stream

---

#### Screen 9: Camera Create/Edit Form
- Form fields (scrollable):
  - Name (required)
  - IP Address (required): monospace input
  - Username: text input
  - Password: password input with eye toggle
  - Tapo Email: email input
  - Tapo Password: password input
  - Location Name: text input
  - Latitude / Longitude: number inputs + "اختر من الخريطة" (Pick from map) button that opens map picker
  - Storage Type: dropdown (SD Card, Cloud, None)
  - Is Active: toggle switch
  - **Remote Access Section** (expandable/collapsible):
    - Connection IP (IP عام للراوتر): text input
    - Connection Port: number input (default 554)
    - API Port: number input (default 443)
    - ONVIF Port: number input (default 2020)
    - Helper text: "اتركها فارغة إذا كانت الكاميرا على نفس الشبكة المحلية"
- **Save button**: full width

---

#### Screen 10: Camera Live View Screen (RTSP Stream)
- **Header**: Camera name + status indicator + "Back" button
- **Full-screen video player area**:
  - Live RTSP stream from camera (using flutter_vlc_player or media_kit)
  - Full-screen toggle button
  - Stream quality indicator (stream1 = HD, stream2 = SD) with toggle
  - Loading spinner while connecting
  - Error state: "تعذر الاتصال بالكاميرا" (Cannot connect to camera) with retry button
- **Camera info bar** below video:
  - Camera name, location, IP address
  - Recording indicator (red dot if camera is recording)
  - Connection status: "متصل" (Connected) in green or "غير متصل" (Disconnected) in red
- **Quick action buttons** below info bar:
  - "التحكم" (Control) → navigates to camera control screen
  - "التسجيلات" (Recordings) → shows recordings list
- **PTZ overlay controls** (optional, semi-transparent on video):
  - D-pad arrows for quick camera movement while viewing
  - Preset quick-access buttons
- Note: RTSP streaming works on Android and iOS only — NOT web

---

#### Screen 11: Camera Control Screen
- **Header**: Camera name + status indicator
- **Live preview button**: "مشاهدة مباشرة" (Live View) button at top → opens live stream screen
- **Control sections** organized as grouped cards:

  - **🚨 Alarm & Security**:
    - Alarm: Start / Stop / Status buttons with toggle state
    - Privacy Mode: ON/OFF toggle
    - LED: ON/OFF toggle
    - Motion Detection: ON/OFF toggle
    - Person Detection: ON/OFF toggle

  - **🎮 Movement & PTZ (Pan-Tilt-Zoom)**:
    - D-pad control (Up / Down / Left / Right arrows around center circle)
    - Speed slider below D-pad
    - Preset buttons: Go To Preset (dropdown), Save Preset, Delete Preset
    - Auto-tracking: ON/OFF toggle
    - Cruise: ON/OFF toggle

  - **⚙️ Camera Settings**:
    - Day/Night mode: Auto / Day / Night selector
    - Flip: ON/OFF toggle
    - Lens Distortion Correction: ON/OFF toggle
    - OSD (On-Screen Display): Show / Hide with text field

  - **💾 Storage & Recording**:
    - SD Card status: capacity bar + format button
    - Recordings: list button → opens recordings list
    - Stream URL: copy button with RTSP URL

  - **🔧 System**:
    - Device Info: model, firmware, MAC address
    - Full Status: expandable detailed status
    - Reboot: button with red confirmation dialog

---

#### Screen 12: Crimes List Screen
- **Filter bar** (horizontal scroll):
  - Status filter: dropdown (All, Pending, Assigned, In Progress, etc.)
  - Severity filter: dropdown (All, Low, Medium, High, Critical)
  - Camera filter: dropdown
  - Officer filter: dropdown
  - Date range picker: "من — إلى"
- **Active filters** shown as removable chips below filter bar
- **Crime cards list** (each card):
  - Severity color strip on left
  - Crime type + icon
  - Status badge (color-coded pill)
  - Camera name
  - Assigned officer name (or "غير مُكلف" if none)
  - Confidence score with mini progress bar
  - Time: creation date/time + time ago
  - Tap → crime detail
- **Sort options**: dropdown (Newest, Oldest, Severity, Distance)
- Pagination: infinite scroll

---

#### Screen 13: Crime Detail Screen (Station View)
- **Map section** (top 35%):
  - Crime location marker (red)
  - Camera location marker (blue)  
  - Assigned officer location (green — real-time if in_progress)
  - Route line between officer and crime (if assigned)
- **Crime info section**:
  - Crime type, severity, status — large display
  - Confidence score: circular gauge
  - Crime date/time
  - Camera name + location
- **Evidence section**:
  - Video player with play/pause, seek, fullscreen
  - Scene time range
- **Assigned Officer section**:
  - Officer avatar, name, rank
  - Status, distance from crime
  - Response time (if arrived)
  - Timeline: assigned → accepted → arrived → resolved (progress steps)
  - Or "لم يتم التكليف بعد" (Not assigned yet) with "تكليف ضابط" button
- **Manual Assignment button** (if unassigned or for reassignment):
  - Opens bottom sheet with list of available officers sorted by distance
  - Each officer shows: name, rank, distance, status
  - Select + confirm → assigns officer
- **Crime status timeline**: vertical stepper showing status history with timestamps

---

#### Screen 14: Chat Conversations List
- **Header**: "المحادثات" (Conversations)
- **Search bar**: "البحث بالاسم..."
- **Conversation list** (sorted by most recent message):
  Each item:
  - Officer avatar circle
  - Officer name (bold)
  - Last message preview (truncated)
  - Timestamp (time ago or date)
  - Unread count badge (red circle with number)
  - Online/offline indicator dot
- Tap → opens chat thread
- Empty state: "لا توجد محادثات"

---

#### Screen 15: Chat Thread Screen
- Same design as officer chat but:
  - Header shows officer name + avatar + rank + online status
  - Station sends from right side (amber)
  - Officer messages on left (gray)
  - Text input area with send button (text messages only, no file/image attachments)

---

#### Screen 16: Profile / Station Settings Screen
- **Station header**:
  - Large avatar with edit overlay
  - Station name (large, bold)
  - Address, city, governorate
- **Editable fields**:
  - Name, Email, Phone, Password change
  - Address, City, Governorate
- **Save button**
- **Logout button** (red, bottom)

---

#### Screen 17: Notifications Screen
- Same design as officer notifications screen but with station-specific notification types:
  - "🚨 تم رصد جريمة في كاميرا المدخل" (Crime detected at entrance camera)
  - "⚠️ نشاط مشبوه — كاميرا الشارع الرئيسي" (Suspicious activity — main street camera)
  - "📍 الضابط أحمد قبل المهمة" (Officer Ahmed accepted the mission)
  - "🔄 تم تصعيد الجريمة #15" (Crime #15 has been escalated)
  - "📷 كاميرا المدخل غير متصلة" (Entrance camera disconnected)
  - "💬 رسالة جديدة من الضابط محمد" (New message from Officer Mohamed)

---

#### Screen 18: Statistics Screen
- **Period selector**: يومي (Daily) / أسبوعي (Weekly) / شهري (Monthly) — segmented control
- **Stats cards**:
  - Total crimes in period
  - Resolved percentage (circular progress)
  - Average response time (in minutes)
  - False alarm rate
- **Charts section**:
  - Crimes over time: line chart
  - Crimes by type: horizontal bar chart
  - Crimes by severity: pie/donut chart
  - Top officers by response time: leaderboard list
- Scrollable, card-based layout
```

---

## Prompt 3: Admin Livewire Dashboard (لوحة تحكم المسؤول)

```
Design a complete web admin dashboard UI for "CrimeLens" — a Crime Prediction System Admin Panel built with Laravel Livewire and Tailwind CSS. This dashboard is used by system administrators to manage police stations, AI models, monitor the entire system, and communicate with stations. The design must be modern, professional, responsive, Arabic-first (RTL) with English support, with a dark sidebar layout.

### Brand & Visual Identity
- App name: "CrimeLens — لوحة التحكم"
- Primary color: Deep Navy Blue (#1B2A4A) — sidebar
- Accent color: Amber/Gold (#F59E0B) — buttons, highlights
- Sidebar: Dark Navy (#0F172A) with lighter active states
- Content area: Light gray background (#F1F5F9)
- Cards: White with subtle shadow
- Typography: Inter/Cairo fonts, clean and professional
- Dark mode supported (toggle in header)
- Responsive: desktop-first, tablet-friendly, mobile-friendly

### Layout Structure:
- **Left Sidebar** (fixed, collapsible):
  - CrimeLens logo at top
  - Navigation items with icons:
    - 📊 لوحة المعلومات (Dashboard)
    - 🏛️ مراكز الشرطة (Police Stations)
    - 🤖 نماذج الذكاء الاصطناعي (AI Models)
    - 💬 المحادثات (Chat)
    - 🔔 الإشعارات (Notifications) — with unread badge
    - ⚙️ الإعدادات (Settings)
  - Collapsed mode: show only icons
  - Active item: lighter background + left accent border
  - Bottom: admin avatar miniature + name + logout icon

- **Top Header Bar** (horizontal):
  - Breadcrumb navigation
  - Search bar (global search)
  - Dark mode toggle
  - Language toggle (AR/EN)
  - Notifications bell with dropdown
  - Admin profile dropdown (name, email, profile link, logout)

### Pages to Design (12 pages total):

---

#### Page 1: Login Page
- Full-page centered login card over gradient background
- CrimeLens logo at top of card
- "لوحة تحكم المسؤول" (Admin Control Panel) subtitle
- Email input with icon
- Password input with eye toggle
- "تسجيل الدخول" (Login) button — amber/gold, full width
- Remember me checkbox
- Clean, minimal design with subtle shadow on card

---

#### Page 2: Dashboard / Overview
- **Welcome banner**: "مرحباً، أحمد" (Welcome, Ahmed) with current date/time
- **4 Metric Cards** (horizontal row):
  1. مراكز الشرطة (Police Stations): total count + icon
  2. الضباط (Officers): total count across all stations + icon
  3. الكاميرات (Cameras): active/total + icon
  4. الجرائم اليوم (Crimes Today): count + comparison with yesterday (↑ or ↓ percentage)
- **Charts Row 1** (2 columns):
  - Left: "الجرائم حسب الحالة" — Donut chart (pending, assigned, resolved, etc.)
  - Right: "اتجاه الجرائم" — Line chart (crimes over last 30 days, daily)
- **Charts Row 2** (2 columns):
  - Left: "الجرائم حسب الخطورة" — Bar chart (low, medium, high, critical)
  - Right: "أداء المراكز" — Horizontal bar chart comparing police stations by crime count
- **Recent Activity section**:
  - Activity log feed (latest 10):
    - Icon + activity description + timestamp
    - e.g. "🚨 تم رصد جريمة جديدة في مركز القاهرة"
    - e.g. "🤖 النموذج AI-1 أرسل heartbeat"
    - e.g. "📷 كاميرا المدخل أصبحت غير متصلة"
- **Active AI Models widget**:
  - Small card showing count of active models with last heartbeat times
  - Red warning if any model missed heartbeat

---

#### Page 3: Police Stations List
- **Header**: "مراكز الشرطة" + "إضافة مركز" button (amber) + "استيراد Excel" button (secondary)
- **Search bar**: "البحث بالاسم أو المدينة..."
- **Table** with columns:
  - # (row number)
  - Avatar (station logo)
  - Name
  - Email
  - Phone
  - City
  - Governorate
  - Officers Count
  - Cameras Count
  - Actions: Edit (icon), Delete (icon), Reset Password (icon), Chat (icon)
- **Pagination** at bottom: showing page numbers + items per page selector
- Table rows hover effect
- Delete: confirmation modal with station name
- Bulk actions: select rows + bulk delete

---

#### Page 4: Police Station Create/Edit Form
- **Card-based form** with sections:
  - **Basic Info Section**:
    - Avatar upload: drag & drop area with preview
    - Name (required)
    - Email (required)
    - Phone
    - Password (create only) / Reset Password button (edit only)
  - **Location Section**:
    - Address (text area)
    - City (text input or dropdown of Egyptian cities)
    - Governorate (dropdown of Egyptian governorates)
  - **Info Section** (edit only, read-only):
    - Number of officers
    - Number of cameras
    - Active crimes count
    - Registration date
- **Action buttons**: "حفظ" (Save) + "إلغاء" (Cancel)
- Inline validation errors in red below each field
- Success toast notification on save

---

#### Page 5: AI Models List
- **Header**: "نماذج الذكاء الاصطناعي" + "إضافة نموذج" button
- **Table** with columns:
  - # (row number)
  - Name
  - Email
  - IP Address (monospace font)
  - Status: "نشط" (Active — green badge) or "غير نشط" (Inactive — red badge)
  - Last Heartbeat: time ago (green if < 2 min, yellow if 2-5 min, red if > 5 min)
  - Assigned Cameras: count with expand icon
  - Actions: Edit, Delete, Reset Password, Toggle Active/Inactive
- **Status indicators**: animated green dot for active models (real-time heartbeat)
- Warning banner if any model has missed heartbeat > 5 minutes

---

#### Page 6: AI Model Create/Edit Form
- **Card-based form**:
  - **Model Info Section**:
    - Name: text input (descriptive label)
    - Email: email input (for login)
    - Password: password input (create) / Reset button (edit)
    - IP Address: text input with validation (IPv4 format)
    - Active: toggle switch
  - **Camera Assignment Section** (CRITICAL):
    - Title: "تعيين الكاميرات" (Assign Cameras)
    - Search/filter cameras
    - **Two-column transfer list** (available ↔ assigned):
      - Left panel: "الكاميرات المتاحة" (Available Cameras) — cameras NOT assigned to any model
      - Right panel: "الكاميرات المعينة" (Assigned Cameras) — cameras assigned to this model
      - Each camera item shows: name, IP, police station name, active status
      - Arrow buttons: → (assign), ← (unassign)
      - Or checkbox list with select all / deselect all
    - Note: "يمكن تعيين كل كاميرا لنموذج واحد فقط" (Each camera can be assigned to one model only)
  - **Model Status Section** (edit only, read-only):
    - Last heartbeat time
    - Is active indicator
    - Assigned cameras count
    - Created date
- **Action buttons**: Save + Cancel

---

#### Page 7: Chat Conversations
- **Two-panel layout** (master-detail):
  - **Left panel** (conversation list, 35% width):
    - Header: "المحادثات" with search bar
    - Conversation list sorted by most recent:
      - Police station avatar
      - Station name
      - Last message preview
      - Timestamp
      - Unread count badge
    - Active conversation highlighted
  - **Right panel** (chat messages, 65% width):
    - Header: station name + avatar + status
    - Message area (scrollable):
      - Admin messages: right side, amber background
      - Station messages: left side, gray background
      - Each: text, timestamp, read receipts
      - Date separators
    - Input area: text field + send button
    - Empty state (no conversation selected): "اختر محادثة للبدء"

---

#### Page 8: Notifications Page
- **Header**: "الإشعارات" + "تحديد الكل كمقروء" button
- **Filter tabs**: الكل (All), غير مقروءة (Unread)
- **Notification cards** (full width list):
  - Icon (type-based) + title + description + timestamp
  - Unread: white card with blue left border
  - Read: slightly gray
  - Notification types:
    - 🚨 "جريمة جديدة في مركز الجيزة" 
    - 🤖 "النموذج AI-2 فقد الاتصال"
    - 📷 "كاميرا الشارع الرئيسي متوقفة"
    - 🏛️ "مركز شرطة جديد تم تسجيله"
    - ⚠️ "تصعيد جريمة — لم يستجب أي ضابط"
- Tap → navigate to relevant section

---

#### Page 9: Settings Page
- **Tabbed interface** or **accordion sections**:
  
  - **🔔 Firebase & Notifications**:
    - Firebase Project ID: text input
    - Firebase Credentials File: file upload
    - Test notification button: send test push
  
  - **🌐 System Settings**:
    - System Language: Arabic / English radio
    - Notification Radius (km): number input (default: 5)
    - Auto-assign crimes: toggle (default: on)
    - Escalation timeout (minutes): number input (default: 5)
    - Max escalation attempts: number input (default: 3)
  
  - **📷 Camera Settings**:
    - Health check interval (minutes): number input (default: 5)
    - Recording segment duration (seconds): number input (default: 60)
    - Recording retention (days): number input (default: 7)
    - Crime evidence retention (days): number input (default: 90)
  
  - **🤖 AI Model Settings**:
    - Heartbeat timeout (minutes): number input (default: 2)
  
  - **📍 Location Settings**:
    - Officer location update distance (meters): number input (default: 100)

- **Save button** at bottom of each section or global save
- Helper text below each setting explaining what it does

---

#### Page 10: Profile Page
- **Profile card**:
  - Large avatar with edit overlay
  - Admin name (large)
  - Email
  - Phone
- **Edit form**:
  - Name, Email, Phone inputs
  - Change Password section (expandable):
    - Current Password
    - New Password
    - Confirm Password
  - Avatar upload with preview
- **Save button**

---

#### Page 11: Police Station Detail (Optional but Recommended)
- **When clicking on a police station name in the table**, show a detail page:
  - Station info header (avatar, name, email, phone, address)
  - **Tabs**: Officers | Cameras | Crimes | Activity
    - Officers tab: mini table of station's officers
    - Cameras tab: mini table of station's cameras with active/inactive status
    - Crimes tab: mini table of recent crimes
    - Activity tab: activity log for this station
  - **Quick Actions**: Reset Password, Send Message, Deactivate

---

#### Page 12: Responsive / Mobile Views
- Design mobile-responsive versions of:
  - Dashboard: cards stack vertically, charts full width
  - Tables: horizontal scroll or card-based list on mobile
  - Sidebar: collapses to hamburger menu
  - Chat: single panel with back navigation
  - Forms: full width, single column
- Show at least iPhone and tablet breakpoints

### Additional UI Elements to Design:

- **Toast Notifications**: Success (green), Error (red), Warning (yellow), Info (blue) — appear top-right
- **Confirmation Modals**: Delete confirmation, Password reset confirmation
- **Loading States**: Skeleton screens for tables and cards
- **Empty States**: Custom illustrations + helpful text for each empty section
- **Error States**: 404 page, 500 page, network error
- **Data Export**: PDF/Excel export buttons on tables and charts
```

---

## ملاحظات عامة (General Notes for All Prompts)

1. **RTL Design**: All text alignment must be right-to-left. Navigation sidebars on the right for web (or standard for mobile). Reading direction is right-to-left.

2. **Arabic Typography**: Use a clean Arabic-compatible font. Ensure all labels, buttons, and text are in Arabic with elegant typography.

3. **Consistency**: All three apps share the same brand identity (CrimeLens, Navy + Amber color scheme). They should look like they belong to the same ecosystem.

4. **Accessibility**: Ensure sufficient color contrast, minimum tap targets (48x48px for mobile), readable font sizes.

5. **Animations**: Indicate where micro-animations should occur (loading, transitions, status changes, alert pulses).

6. **State Variations**: For each interactive element, show all states (default, hover, active, disabled, loading, error, success).

7. **Language Control**: Mobile apps (Officer + Police Station) do NOT have a language switch — the language is controlled by the admin from the dashboard only. The app fetches the system language from `GET /api/v1/settings/language` on startup. Only the admin dashboard has a language toggle.

8. **Encryption**: Camera-related data (RTSP URLs, credentials) is encrypted on the server and decrypted on the client. The Flutter apps receive an encryption key at login. Design-wise, this is invisible to the user — no UI elements needed for encryption.

9. **Live Streaming**: Police station app supports live RTSP camera viewing via `flutter_vlc_player` or `media_kit`. This feature is Android/iOS only — show a "not supported" message on web.
