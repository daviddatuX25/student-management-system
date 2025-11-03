Student Management System - Business Logic & Rules Guide
📜 Single Source of Truth for Business Rules, Flows & Processes

🔐 1. Authentication & Authorization
1.1 User Authentication Rules
Student Login:
RULE: Students must login with Student ID (not email)
Format: E[YY]-[NNNNN]
Example: E23-00345

Process:
1. User enters Student ID in login form
2. System looks up user where student_id = input AND role = 'student'
3. If found, verify password
4. If valid, create session with role = 'student'
5. Redirect to student.dashboard

Validation:
- Student ID must match pattern: /^E\d{2}-\d{5}$/
- Student ID is case-insensitive
- Account must have status = 'active'

Error Cases:
- Invalid format → "Invalid Student ID format"
- Student not found → "Invalid credentials"
- Inactive account → "Your account has been deactivated. Contact admin."
Admin/Teacher Login:
RULE: Admin and Teachers login with email

Process:
1. User enters email + password
2. System verifies credentials
3. Check role (admin/teacher)
4. Redirect based on role:
   - admin → admin.dashboard
   - teacher → teacher.dashboard

Validation:
- Email must be valid format
- Account must be active
1.2 Authorization Matrix
ActionAdminTeacherStudentView all students✅❌❌Create student✅❌❌Edit student✅❌❌Delete student✅❌❌View all classes✅❌❌Create class✅❌❌Edit class✅❌❌Delete class✅❌❌Assign student to class✅Request Only❌View own classes✅✅✅Mark attendance✅✅ (own classes)❌View all attendance✅✅ (own classes)✅ (own only)Manage rooms✅❌❌View notifications✅✅✅
Implementation:
php// In controllers or middleware
if (!auth()->user()->isAdmin()) {
    abort(403, 'Unauthorized action');
}

// For teacher-specific class access
if (!auth()->user()->isTeacher() || $class->teacher_id !== auth()->id()) {
    abort(403, 'You can only access your own classes');
}

👨‍🎓 2. Student Management Rules
2.1 Student ID Generation
RULE: Auto-generate unique Student ID on creation
Format: E[YY]-[NNNNN]
- E: Prefix (fixed)
- YY: Enrollment year (last 2 digits)
- NNNNN: Sequential 5-digit number

Example: E23-00345 (enrolled in 2023, 345th student)

Algorithm:
1. Get current enrollment year (from enrollment_date)
2. Extract last 2 digits: year = enrollment_date->format('y')
3. Find highest number for that year:
   SELECT MAX(CAST(SUBSTRING(student_id, 5) AS UNSIGNED)) 
   FROM users 
   WHERE student_id LIKE 'E{year}-%'
4. Increment by 1, pad to 5 digits
5. Concatenate: "E{year}-{padded_number}"

Edge Cases:
- First student of year → E23-00001
- 99,999th student → E23-99999 (then consider format change or error)
- Same enrollment date for multiple students → Sequential assignment

Implementation Location:
- Model: User::generateStudentId($enrollmentDate)
- Called in: StudentController@store
2.2 Student Creation Rules
Who Can Create:

✅ Admin only
❌ Teachers cannot create students
❌ Public registration is disabled

Required Fields:
MANDATORY:
- name (string, max 255)
- email (unique, valid email format)
- enrollment_date (date, cannot be future date)

OPTIONAL:
- photo (image, max 2MB, formats: jpg, jpeg, png)
- date_of_birth (date, must be at least 10 years ago)
- address (text)
- phone (string, valid phone format)

AUTO-GENERATED:
- student_id (generated via algorithm)
- password (temporary, format: StudentID + DOB)
- status (default: 'active')
- role (default: 'student')
Password Generation:
RULE: Temporary password = StudentID (no dashes) + MMDDYYYY of DOB

Example:
- Student ID: E23-00345
- DOB: 2005-03-15
- Password: E2300345 + 03152005 = E2300345031520­05

Student must change on first login (upcoming feature)
Validation Rules:
php// StudentStoreRequest
public function rules() {
    return [
        'name' => 'required|string|max:255',
        'email' => 'required|email|unique:users,email',
        'enrollment_date' => 'required|date|before_or_equal:today',
        'date_of_birth' => 'nullable|date|before:-10 years',
        'address' => 'nullable|string|max:500',
        'phone' => 'nullable|string|regex:/^[0-9]{10,15}$/',
        'photo' => 'nullable|image|mimes:jpg,jpeg,png|max:2048',
    ];
}
Business Logic Flow:
1. Admin fills create student form
2. Validate all inputs
3. IF validation passes:
   a. Generate student_id
   b. Generate temporary password
   c. Hash password
   d. Store photo (if uploaded) → storage/app/public/photos
   e. Create user record with role='student', status='active'
   f. TRIGGER: StudentCreated event (upcoming feature)
   g. Redirect to student.show with success message
4. ELSE:
   a. Return to form with validation errors
2.3 Student Update Rules
Editable Fields:
ALLOWED TO EDIT:
- name
- email (must remain unique)
- date_of_birth
- address
- phone
- photo (replace existing)
- status (active/inactive)

NOT EDITABLE:
- student_id (immutable)
- enrollment_date (historical record)
- role (always 'student')
Status Change Impact:
IF status changed from 'active' to 'inactive':
- Student cannot login
- Still appears in historical records
- Still visible in class rosters (with inactive badge)
- Cannot be assigned to new classes
- Does NOT auto-remove from existing classes

IF status changed from 'inactive' to 'active':
- Student can login again
- Can be assigned to new classes
2.4 Student Deletion Rules
RULE: Soft Delete Only
Process:
1. Admin clicks delete
2. Confirmation modal: "Are you sure? This will remove the student from all classes."
3. IF confirmed:
   a. Remove all class assignments (delete from student_class pivot)
   b. Soft delete user (deleted_at = now())
   c. TRIGGER: StudentDeleted event
   d. Notify affected teachers
   e. Return to students.index with success message

Constraints:
- Cannot hard delete (preserve data integrity)
- Can restore via database if needed (admin portal feature upcoming)

Cascade Effects:
- Attendance records: KEEP (historical data)
- Class assignments: REMOVE
- Notifications: KEEP (archived)

📚 3. Class Management Rules
3.1 Class Creation Rules
Required Fields:
MANDATORY:
- class_code (unique, alphanumeric, max 20 chars)
  Example: SIA-Y3-A, MATH101-A
- class_name (string, max 255)
  Example: "System Integration Architecture"
- year_level (integer, 1-6 for college, 1-12 for K-12)
- section (string, max 10)
  Example: "A", "B1", "Advanced"
- capacity (integer, min: 1, max: 200)
- room_id (must exist in rooms table)
- teacher_id (must exist in users table with role='teacher')
- schedules (array, min: 1 schedule)

OPTIONAL:
- description (text)
- status (default: 'active')

AUTO-GENERATED:
- None (all fields explicit)
Schedule Structure:
EACH SCHEDULE MUST HAVE:
- day_of_week (enum: monday, tuesday, wednesday, thursday, friday, saturday, sunday)
- start_time (time format: HH:MM)
- end_time (time format: HH:MM)

Example:
{
    "schedules": [
        {
            "day_of_week": "monday",
            "start_time": "08:00",
            "end_time": "10:00"
        },
        {
            "day_of_week": "wednesday",
            "start_time": "08:00",
            "end_time": "10:00"
        }
    ]
}
Validation Rules:
1. Class Code:
   - Must be unique across all classes
   - Alphanumeric + hyphens only
   - Recommended format: [SUBJECT]-Y[YEAR]-[SECTION]

2. Capacity:
   - Must be >= current enrolled count
   - If editing, cannot set below current enrollment

3. Teacher Assignment:
   - Teacher must have role='teacher'
   - Teacher must be active status
   - Teacher can teach multiple classes (no limit)

4. Room Assignment:
   - Room must exist
   - Room availability checked (no double-booking)

5. Schedule Validation:
   - end_time must be > start_time
   - Minimum class duration: 30 minutes
   - Maximum class duration: 4 hours
   - No overlapping schedules for same room
   - No overlapping schedules for same teacher
3.2 Room Conflict Detection
RULE: A room cannot be double-booked
Algorithm:
1. When creating/editing class with schedules
2. For each schedule:
   a. Query existing schedules for same room_id
   b. Filter by same day_of_week
   c. Check time overlap:
      IF (new_start < existing_end AND new_end > existing_start):
          → CONFLICT DETECTED
3. If any conflict found:
   - Return error with conflicting class details
   - Do not save class

Implementation:
public function hasRoomConflict($roomId, $dayOfWeek, $startTime, $endTime, $excludeClassId = null) {
    return Schedule::whereHas('class', function($query) use ($roomId) {
        $query->where('room_id', $roomId);
    })
    ->where('day_of_week', $dayOfWeek)
    ->where(function($query) use ($startTime, $endTime) {
        $query->where(function($q) use ($startTime, $endTime) {
            $q->where('start_time', '<', $endTime)
              ->where('end_time', '>', $startTime);
        });
    })
    ->when($excludeClassId, function($query, $id) {
        $query->where('class_id', '!=', $id);
    })
    ->exists();
}
3.3 Teacher Conflict Detection
RULE: A teacher cannot teach two classes at the same time
Algorithm: Same as room conflict, but check teacher_id instead of room_id

Edge Cases:
- Teacher can teach multiple sections of same subject (different times)
- Teacher can teach back-to-back classes (e.g., 8-10am, 10-12pm)
- Teacher schedule can have gaps (free periods)

Error Message:
"Prof. John Doe is already teaching [Class Name] on [Day] from [Start] to [End]"
3.4 Class Deletion Rules
RULE: Soft delete with student de-enrollment
Process:
1. Admin clicks delete class
2. Check if class has enrolled students
3. IF has students:
   - Show confirmation: "This class has X enrolled students. They will be removed from this class. Continue?"
4. IF confirmed:
   a. Remove all student assignments (student_class pivot)
   b. Soft delete class (deleted_at = now())
   c. Soft delete associated schedules
   d. TRIGGER: ClassDeleted event
   e. Notify all affected students and teacher
5. ELSE:
   - Cancel deletion

Constraints:
- Cannot delete if class has active attendance records for today
- Can restore via admin panel (upcoming feature)

Cascade Effects:
- Schedules: SOFT DELETE
- Student assignments: HARD DELETE (pivot table)
- Attendance records: KEEP (historical)
- Notifications: KEEP (archived)

🎯 4. Class Assignment Rules
4.1 Assignment Prerequisites
Before assigning student to class, verify:
1. ✅ Student exists and is active
2. ✅ Class exists and is active
3. ✅ Class has available capacity
4. ✅ Student is not already enrolled
5. ✅ No schedule conflicts
6. ✅ Assigner has permission (admin or approved request)
4.2 Capacity Check
RULE: Cannot exceed class capacity
Algorithm:
1. Get class.capacity
2. Count current enrolled students:
   enrolled = class->students()->count()
3. IF enrolled >= capacity:
   → REJECT with error "Class is full (X/X students)"
4. ELSE:
   → PROCEED to next check

Display Format:
"Mathematics 101-A (35/40 students)" 
→ Shows 5 slots available
4.3 Schedule Conflict Detection (Critical)
RULE: A student cannot be in two classes at the same time
This is the MOST IMPORTANT validation rule in the system.

Algorithm:
1. Get all schedules for the target class
2. Get all classes the student is currently enrolled in
3. For each enrolled class:
   a. Get all its schedules
   b. For each schedule, check against target class schedules:
      - Same day_of_week?
      - Time overlap?
4. If ANY conflict found:
   → REJECT with detailed error message

Time Overlap Logic:
IF (schedule1.day == schedule2.day) AND
   (schedule1.start < schedule2.end) AND
   (schedule1.end > schedule2.start):
   → CONFLICT

Example Scenarios:

✅ NO CONFLICT:
- Class A: Monday 8:00-10:00
- Class B: Monday 10:00-12:00
(Back-to-back is allowed)

❌ CONFLICT:
- Class A: Monday 8:00-10:00
- Class B: Monday 9:00-11:00
(1 hour overlap)

❌ CONFLICT:
- Class A: Monday 8:00-12:00
- Class B: Monday 9:00-10:00
(Class B entirely within Class A)

✅ NO CONFLICT:
- Class A: Monday 8:00-10:00
- Class B: Tuesday 8:00-10:00
(Different days)
Conflict Error Message Format:
"Schedule conflict detected:
- Existing: English 101-A (Monday 9:00-11:00)
- New: Mathematics 101-A (Monday 10:00-12:00)
Time overlap: 1 hour"
4.4 Single Assignment Flow
Process for Admin:
1. Admin navigates to admin.assignments.index
2. Selects student from dropdown (searchable)
3. Selects class from dropdown (shows capacity)
4. Clicks "Check Conflicts"
5. System validates:
   a. Capacity check
   b. Already enrolled check
   c. Schedule conflict check
6. IF all validations pass:
   - Show success preview: "Ready to assign [Student] to [Class]"
   - Click "Confirm Assignment"
   - System creates record in student_class pivot:
     {
       student_id: X,
       class_id: Y,
       enrolled_at: now(),
       assigned_by: auth()->id(),
       status: 'active'
     }
   - TRIGGER: StudentEnrolled event
   - Show success message
7. ELSE:
   - Show validation errors
   - Do NOT create assignment
Process for Teacher (Request):
1. Teacher navigates to their class page
2. Clicks "Request Student Assignment"
3. Selects student
4. System creates assignment_request:
   {
     teacher_id: auth()->id(),
     student_id: X,
     class_id: Y,
     status: 'pending',
     requested_at: now()
   }
5. TRIGGER: TeacherAssignmentRequested event
6. Admin receives notification
7. Admin reviews request in admin.assignments.requests
8. Admin can:
   a. Approve → Run full validation → Assign
   b. Reject → Update request status → Notify teacher

Status Flow:
pending → approved → active (student enrolled)
pending → rejected → closed
4.5 Bulk Assignment Flow
RULE: Bulk assign with comprehensive conflict reporting
Process:
1. Admin uploads CSV or selects multiple students
2. Admin selects target class
3. System validates each student individually
4. Generate detailed report:

Report Structure:
{
    "total_students": 50,
    "successful": [
        {student_id: 1, name: "John Doe"},
        {student_id: 2, name: "Jane Smith"},
        ...
    ],
    "failed": [
        {
            student_id: 3,
            name: "Bob Johnson",
            reason: "schedule_conflict",
            details: {
                conflict_class: "English 101-A",
                conflict_schedule: "Monday 9:00-11:00",
                overlap_duration: "1 hour"
            }
        },
        {
            student_id: 4,
            name: "Alice Brown",
            reason: "capacity_exceeded",
            details: "Class is full (40/40 students)"
        }
    ],
    "success_count": 47,
    "failure_count": 3
}

5. Display report to admin
6. Admin can:
   - Proceed with successful assignments only
   - Download failure report for manual review
   - Cancel entire operation

Implementation:
foreach ($studentIds as $studentId) {
    try {
        $this->validateAssignment($studentId, $classId);
        $this->assignStudent($studentId, $classId);
        $results['successful'][] = $student;
    } catch (ValidationException $e) {
        $results['failed'][] = [
            'student' => $student,
            'reason' => $e->getMessage(),
            'details' => $e->getDetails()
        ];
    }
}
4.6 Student Removal from Class
RULE: Can remove student from class if not currently in session
Process:
1. Admin/Teacher navigates to class roster
2. Clicks "Remove" next to student name
3. Confirmation modal:
   "Remove [Student Name] from [Class Name]?"
4. IF confirmed:
   a. Check if class is currently active (based on schedule)
   b. IF class in session:
      → ERROR: "Cannot remove student during active class"
   c. ELSE:
      - Delete record from student_class pivot
      - TRIGGER: StudentRemoved event
      - Show success message
      - Notify student and teacher

Constraints:
- Cannot remove if attendance is being taken right now
- Can remove if class has past attendance records (keeps historical data)
- Can remove even if student has pending assignments (upcoming feature)

Status Update:
- Instead of deleting, update pivot status to 'dropped'
- Keep record for historical tracking
- Filter out 'dropped' in active enrollments

📋 5. Attendance Management Rules
5.1 Attendance Status Definitions
STATUSES:

1. PRESENT
   - Student attended on time
   - Check-in within grace period (0-15 minutes after start)
   - Color: Green

2. LATE
   - Student attended after grace period
   - Check-in > 15 minutes after start time
   - Still counts as attended but flagged
   - Color: Yellow/Orange

3. ABSENT
   - Student did not attend
   - No check-in recorded
   - Auto-marked if session ends
   - Color: Red

4. EXCUSED
   - Student did not attend but has valid reason
   - Only settable by teacher (manual override)
   - Examples: sick leave, school event, emergency
   - Color: Blue/Gray
5.2 Grace Period Logic
RULE: 15-minute grace period after class start time
Algorithm:
1. Get schedule.start_time for the class session
2. When student checks in (face recognition or manual):
   check_in_time = now()
3. Calculate difference:
   diff_minutes = check_in_time - schedule.start_time
4. Determine status:
   IF diff_minutes <= 15:
       status = 'present'
   ELSE:
       status = 'late'
5. Store attendance record with check_in_time

Example Scenarios:

Class Schedule: Monday 8:00 AM - 10:00 AM
Grace Period Ends: 8:15 AM

- Student arrives 7:55 AM → PRESENT (early)
- Student arrives 8:00 AM → PRESENT (on time)
- Student arrives 8:15 AM → PRESENT (within grace)
- Student arrives 8:16 AM → LATE (1 min past grace)
- Student arrives 9:30 AM → LATE (during class)
- Student arrives 10:01 AM → ABSENT (class ended, manual mark only)

Configuration:
- Grace period: 15 minutes (configurable in config/student.php)
- Can be adjusted per class if needed (upcoming feature)
5.3 Manual Attendance Marking
Who Can Mark:

✅ Teacher (for their own classes)
✅ Admin (for any class)
❌ Students (cannot mark own attendance)

Process:
1. Teacher navigates to teacher.attendance.create
2. Selects class and schedule (today's schedule)
3. System loads all enrolled students for that class
4. For each student, teacher selects:
   - Present
   - Absent
   - Late
   - Excused
5. Optional: Add notes per student
6. Click "Save Attendance"
7. System validates:
   - Attendance not already marked for this date+class+student
   - Teacher has permission for this class
8. IF valid:
   - Create attendance records (bulk insert)
   - TRIGGER: AttendanceMarked event for each student
   - Show success message
9. ELSE:
   - Show errors

Constraints:
- Can only mark attendance for current or past dates
- Cannot mark future attendance
- Cannot mark attendance twice for same student+class+date
- Can edit attendance within 24 hours (upcoming feature)
5.4 Face Recognition Attendance Flow
This is the UNIQUE FEATURE - Critical Implementation
OVERVIEW:
Teacher places phone/tablet at classroom entrance with camera facing students.
Students walk by, system detects faces and marks attendance automatically.

PREREQUISITES:
- Student photos uploaded during registration
- Face detection API/library integrated (e.g., face-api.js, AWS Rekognition)
- Mobile browser with camera permission

PROCESS:

1. SESSION INITIALIZATION
   - Teacher navigates to teacher.attendance.face-recognition
   - Selects class and schedule
   - System validates:
     • Class scheduled for now (within 30 min window)
     • No attendance session already active
   - Click "Start Face Recognition Session"
   - System creates session record:
     {
       class_id: X,
       schedule_id: Y,
       teacher_id: auth()->id(),
       status: 'active',
       started_at: now(),
       grace_period_ends_at: now()->addMinutes(15)
     }

2. CAMERA ACTIVATION
   - Mobile browser requests camera permission
   - Stream video to canvas element
   - Load face detection model
   - Start continuous detection loop (every 2 seconds)

3. FACE DETECTION LOOP
   While session is active:
   a. Capture frame from video stream
   b. Detect faces in frame
   c. For each detected face:
      - Extract face descriptor (embedding)
      - Compare with stored student photos
      - Match threshold: 80% similarity
   d. IF match found AND student enrolled in class:
      - Check if already marked today
      - IF not marked:
        * Determine status (present/late based on grace period)
        * Create attendance record
        * TRIGGER: AttendanceMarked event
        * Visual feedback: Green check + student name on screen
        * Audio feedback: Beep sound
      - ELSE:
        * Visual feedback: Yellow "Already marked"
   e. IF match not found OR not enrolled:
      - Visual feedback: Red "Not recognized / Not enrolled"

4. GRACE PERIOD MONITORING
   - Display countdown timer on screen
   - When grace_period_ends_at is reached:
     • Status changes from 'present' to 'late' for new check-ins
     • Visual indicator changes (yellow border)

5. SESSION TERMINATION
   Teacher clicks "Stop Session"
   OR
   Auto-stop after class end_time + 15 minutes
   
   Process:
   - Update session status to 'completed'
   - Stop face detection loop
   - Release camera
   - Auto-mark ABSENT for non-checked students:
     enrolled_students = class.students
     checked_students = attendances.where(date = today, class_id = X)
     absent_students = enrolled_students - checked_students
     foreach absent_students:
       create attendance(status: 'absent', method: 'auto')
   - Generate attendance summary report
   - TRIGGER: AttendanceSessionCompleted event

6. TEACHER MANUAL OVERRIDE
   After session ends:
   - Teacher can view attendance list
   - Edit any record:
     • Change ABSENT to EXCUSED (with note)
     • Change LATE to PRESENT (with justification)
     • Change PRESENT to ABSENT (if mistaken detection)
   - Each override logged with:
     {
       overridden_by: teacher_id,
       overridden_at: timestamp,
       old_status: 'late',
       new_status: 'present',
       reason: 'Student provided medical certificate'
     }
Face Recognition Technical Details:
Library: face-api.js (client-side, privacy-friendly)

Setup:
1. Load models:
   - face-api.nets.tinyFaceDetector.loadFromUri('/models')
   - face-api.nets.faceLandmark68Net.loadFromUri('/models')
   - face-api.nets.faceRecognitionNet.loadFromUri('/models')

2. Process student photos on registration:
   - Detect face in uploaded photo
   - Extract 128-dimension descriptor
   - Store in database: student_face_descriptors table
   {
     student_id: X,
     descriptor: [0.123, -0.456, ...] // JSON array
   }

3. Real-time matching:
   detectedDescriptor = await faceapi.detectSingleFace(frame)
       .withFaceLandmarks()
       .withFaceDescriptor()
   
   foreach storedDescriptor in database:
       distance = faceapi.euclideanDistance(detectedDescriptor, storedDescriptor)
       IF distance < 0.6: // Threshold for match
           → Student matched

4. Optimization:
   - Process every 2 seconds (not every frame)
   - Use TinyFaceDetector for speed
   - Limit detection area (center of frame)
   - Cache student descriptors in memory during session
Error Handling:
1. Camera permission denied:
   → Fallback to manual attendance marking
   → Show error message with instructions

2. Poor lighting:
   → Display warning: "Lighting too dark for face detection"
   → Suggest moving to better lit area

3. Multiple faces detected:
   → Process each face independently
   → Show count on screen: "Detecting 3 faces..."

4. No face detected (student wearing mask):
   → Allow teacher to manually mark
   → Or student shows QR code (alternative, upcoming feature)

5. Network issues:
   → Queue attendance records locally
   → Sync when connection restored
   → Show "Offline mode" indicator
5.5 Attendance Reports & Queries
Common Queries:
1. Student Attendance Rate
   SELECT 
       COUNT(CASE WHEN status IN ('present', 'late') THEN 1 END) * 100.0 / COUNT(*) as rate
   FROM attendances
   WHERE student_id = X AND date >= start_of_semester

2. Class Attendance Today
   SELECT students.name, attendances.status, attendances.check_in_time
   FROM attendances
   JOIN users as students ON attendances.student_id = students.id
   WHERE attendances.class_id = X AND attendances.date = CURDATE()

3. Students with Low Attendance (<75%)
   [Complex query with subqueries and aggregation]

4. Most Punctual Students (never late)
   SELECT student_id, COUNT(*) as present_count
   FROM attendances
   WHERE status = 'present' AND check_in_time <= grace_period
   GROUP BY student_id
   HAVING COUNT(CASE WHEN status = 'late' THEN 1 END) = 0

🔔 6. Notification System Rules
6.1 Notification Triggers
Event → Notification Matrix:
EventRecipientsNotification TypePriorityStudentEnrolledStudent, Teacher, AdminIn-app + BrowserMediumStudentRemovedStudent, Teacher, AdminIn-app + BrowserHighAttendanceMarked (Late/Absent)StudentIn-app + BrowserMediumClassScheduleChangedAll enrolled students, TeacherIn-app + BrowserHighTeacherAssignmentRequestedAdminIn-app + BrowserLowClassCapacityFullAdmin, TeacherIn-appLowAttendanceSessionStartedAll enrolled studentsBrowser (push)High
6.2 Notification Delivery Rules
In-App Notifications:
Storage: notifications table (Laravel default)
Retention: Unlimited (user can delete)
Display: Dropdown in navbar (max 10 recent, link to view all)
Mark as Read: User clicks notification or "Mark all as read"

Structure:
{
    "type": "enrollment",
    "title": "New Class Enrollment",
    "message": "You have been enrolled in Mathematics 101-A",
    "data": {
        "class_id": 5,
        "class_name": "Mathematics 101-A",
        "action_url": "/student/classes/5"
    },
    "read_at": null,
    "created_at": "2024-11-03 10:30:00"
}
Browser Push Notifications:
Technology: Laravel Echo + Pusher (or Socket.io)
Permission: Request on first login
Display: Native browser notification
Behavior:
- Only sent if user has enabled browser notifications
- Only sent if page is not in focus
- Auto-dismiss after 10 seconds
- Click → Navigate to relevant page

Implementation:
// Listen for broadcast event
Echo.private('user.' + userId)
    .notification((notification