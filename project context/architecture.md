# Student Management System - Complete Architecture Blueprint

## 🎯 System Overview

**Tech Stack:**
- Laravel 10.x
- Livewire 3.x
- Tailwind CSS
- SQLite (dev) / MySQL (prod)
- Laravel Events & Notifications

**Unique Feature:** 🌟 **Face Recognition Attendance System** (Mobile-based camera scanning at classroom entrance)

---

## 📊 Database Schema

### Users Table (Laravel Default + Extended)
```
users
├── id (PK)
├── name
├── email (unique)
├── password
├── role (enum: admin, teacher, student)
├── student_id (nullable, unique) - Format: E23-00345
├── photo (nullable)
├── date_of_birth (nullable)
├── address (nullable)
├── phone (nullable)
├── status (enum: active, inactive) - default: active
├── enrollment_date (nullable)
├── timestamps
└── soft_deletes
```

### Classes Table
```
classes
├── id (PK)
├── class_code (unique) - e.g., "SIA-Y3-A"
├── class_name - e.g., "System Integration Architecture"
├── description (text)
├── year_level (integer)
├── section
├── capacity (integer)
├── room_id (FK)
├── teacher_id (FK) - references users
├── status (enum: active, inactive, archived)
├── timestamps
└── soft_deletes
```

### Rooms Table
```
rooms
├── id (PK)
├── room_code (unique) - e.g., "BLDG-A-301"
├── room_name
├── building (nullable)
├── capacity (integer)
├── timestamps
```

### Schedules Table
```
schedules
├── id (PK)
├── class_id (FK)
├── day_of_week (enum: monday, tuesday, wednesday, thursday, friday, saturday, sunday)
├── start_time (time)
├── end_time (time)
├── timestamps
```

### Student_Class (Pivot Table)
```
student_class
├── id (PK)
├── student_id (FK) - references users
├── class_id (FK)
├── enrolled_at (timestamp)
├── status (enum: active, dropped, completed)
├── assigned_by (FK) - references users (admin)
├── timestamps
```

### Attendances Table
```
attendances
├── id (PK)
├── student_id (FK)
├── class_id (FK)
├── schedule_id (FK)
├── date
├── status (enum: present, absent, late, excused)
├── check_in_time (nullable)
├── marked_by (FK) - references users (teacher/system)
├── method (enum: face_recognition, manual)
├── notes (nullable, text)
├── timestamps
```

### Notifications Table (Laravel Default)
```
notifications
├── id (UUID, PK)
├── type
├── notifiable_type
├── notifiable_id
├── data (JSON)
├── read_at (nullable)
└── created_at
```

---

## 🏗️ Models & Relationships

### User Model
**Location:** `app/Models/User.php`

```php
Relationships:
- hasMany(Class) - as teacher
- belongsToMany(Class, 'student_class') - as student
- hasMany(Attendance)
- hasMany(Notification)

Scopes:
- scopeStudents()
- scopeTeachers()
- scopeAdmins()
- scopeActive()

Accessors:
- getFullNameAttribute()
- getPhotoUrlAttribute()

Methods:
- isAdmin(), isTeacher(), isStudent()
- hasScheduleConflict($classId)
- generateStudentId() - static
```

### Class Model
**Location:** `app/Models/ClassModel.php` (avoid naming conflict with PHP keyword)

```php
Relationships:
- belongsTo(User, 'teacher_id')
- belongsTo(Room)
- hasMany(Schedule)
- belongsToMany(User, 'student_class')->withPivot('enrolled_at', 'status')
- hasMany(Attendance)

Scopes:
- scopeActive()
- scopeWithAvailableSlots()

Accessors:
- getFullNameAttribute() - "SIA Year 3 Section A"
- getEnrolledCountAttribute()
- getAvailableSlotsAttribute()

Methods:
- hasCapacity()
- hasScheduleConflict($studentId)
- getScheduleDetails()
```

### Room Model
**Location:** `app/Models/Room.php`

```php
Relationships:
- hasMany(Class)

Methods:
- isAvailable($dayOfWeek, $startTime, $endTime)
```

### Schedule Model
**Location:** `app/Models/Schedule.php`

```php
Relationships:
- belongsTo(Class)

Methods:
- conflictsWith(Schedule $otherSchedule)
- isActiveNow()
```

### Attendance Model
**Location:** `app/Models/Attendance.php`

```php
Relationships:
- belongsTo(User, 'student_id')
- belongsTo(Class)
- belongsTo(Schedule)
- belongsTo(User, 'marked_by')

Scopes:
- scopeForDate($date)
- scopeForClass($classId)
- scopePresent(), scopeAbsent(), scopeLate()

Methods:
- isLate() - checks if check_in_time > schedule start_time + 15 min
```

---

## 🎮 Controllers

### Admin Controllers

**AdminDashboardController**
- `index()` - Show admin dashboard with stats

**StudentController**
- `index()` - List all students (with search/filter)
- `create()` - Show create form
- `store(StudentStoreRequest)` - Create student
- `show($id)` - Student profile
- `edit($id)` - Edit form
- `update($id, StudentUpdateRequest)` - Update student
- `destroy($id)` - Soft delete student
- `bulkImport()` - CSV import view
- `processBulkImport(Request)` - Process CSV

**ClassController**
- `index()` - List all classes
- `create()` - Create form
- `store(ClassStoreRequest)` - Create class
- `show($id)` - Class details with enrolled students
- `edit($id)` - Edit form
- `update($id, ClassUpdateRequest)` - Update class
- `destroy($id)` - Delete class
- `roster($id)` - View/export class roster PDF

**ClassAssignmentController**
- `index()` - Assignment management view
- `assign(AssignStudentRequest)` - Assign student to class
- `bulkAssign(BulkAssignRequest)` - Bulk assignment with conflict check
- `remove($studentId, $classId)` - Remove student from class

**RoomController**
- `index()` - List rooms
- `store(RoomRequest)` - Create room
- `update($id, RoomRequest)` - Update room
- `destroy($id)` - Delete room

### Teacher Controllers

**TeacherDashboardController**
- `index()` - Teacher dashboard with their classes

**TeacherClassController**
- `index()` - List teacher's classes
- `show($id)` - Class details with students

**AttendanceController**
- `index($classId)` - View attendance records
- `create($classId, $scheduleId)` - Start attendance session
- `store(AttendanceRequest)` - Manual attendance marking
- `update($id, AttendanceRequest)` - Update attendance record
- `startFaceRecognition($classId, $scheduleId)` - Initialize face scan session
- `recordFaceAttendance(Request)` - API endpoint for face recognition

**AssignmentRequestController**
- `store(Request)` - Teacher requests student assignment (pending approval)

### Student Controllers

**StudentDashboardController**
- `index()` - Student dashboard with their classes

**StudentClassController**
- `index()` - View enrolled classes
- `show($id)` - View class details

**StudentAttendanceController**
- `index()` - View own attendance records

### Auth Controllers (Laravel Breeze/Fortify)
- Custom login to handle student_id authentication
- Standard registration (admin creates accounts, so disable public registration)

---

## 📝 Form Requests

**StudentStoreRequest**
```php
Rules:
- name: required|string|max:255
- email: required|email|unique:users
- date_of_birth: nullable|date
- address: nullable|string
- phone: nullable|string
- photo: nullable|image|max:2048
- enrollment_date: required|date
```

**StudentUpdateRequest**
```php
Rules: Same as Store, but email unique ignores current user
```

**ClassStoreRequest**
```php
Rules:
- class_code: required|unique:classes
- class_name: required|string
- year_level: required|integer
- section: required|string
- capacity: required|integer|min:1
- room_id: required|exists:rooms,id
- teacher_id: required|exists:users,id (must be teacher)
- schedules: required|array (nested validation for day/time)
```

**AssignStudentRequest**
```php
Rules:
- student_id: required|exists:users,id
- class_id: required|exists:classes,id

Custom Validation:
- checkCapacity()
- checkScheduleConflict()
- checkAlreadyEnrolled()
```

**BulkAssignRequest**
```php
Rules:
- students: required|array
- class_id: required|exists:classes,id

Custom Validation:
- validateBulkConflicts()
```

**AttendanceRequest**
```php
Rules:
- student_id: required|exists:users,id
- class_id: required|exists:classes,id
- status: required|in:present,absent,late,excused
- notes: nullable|string
```

---

## 🎨 Blade Views Structure

```
resources/views/
├── layouts/
│   ├── app.blade.php (main layout with Livewire)
│   ├── admin.blade.php
│   ├── teacher.blade.php
│   └── student.blade.php
│
├── auth/
│   ├── login.blade.php (custom for student_id)
│   └── forgot-password.blade.php
│
├── admin/
│   ├── dashboard.blade.php
│   ├── students/
│   │   ├── index.blade.php (with Livewire search)
│   │   ├── create.blade.php
│   │   ├── edit.blade.php
│   │   ├── show.blade.php
│   │   └── bulk-import.blade.php
│   │
│   ├── classes/
│   │   ├── index.blade.php
│   │   ├── create.blade.php (with schedule builder)
│   │   ├── edit.blade.php
│   │   ├── show.blade.php
│   │   └── roster.blade.php (PDF export)
│   │
│   ├── assignments/
│   │   ├── index.blade.php (Livewire component)
│   │   └── bulk-assign.blade.php
│   │
│   └── rooms/
│       └── index.blade.php
│
├── teacher/
│   ├── dashboard.blade.php
│   ├── classes/
│   │   ├── index.blade.php
│   │   └── show.blade.php
│   │
│   └── attendance/
│       ├── index.blade.php
│       ├── mark.blade.php (manual marking)
│       └── face-recognition.blade.php (camera interface)
│
├── student/
│   ├── dashboard.blade.php
│   ├── classes/
│   │   ├── index.blade.php (schedule calendar view)
│   │   └── show.blade.php
│   │
│   └── attendance/
│       └── index.blade.php
│
└── components/
    ├── notification-dropdown.blade.php
    ├── schedule-calendar.blade.php
    ├── student-card.blade.php
    └── class-card.blade.php
```

---

## ⚡ Livewire Components

### Admin Components

**StudentSearchTable** (`app/Livewire/Admin/StudentSearchTable.php`)
```php
Properties:
- $search
- $status (filter)
- $perPage

Methods:
- render() - with pagination
- updatingSearch() - reset pagination
- exportExcel()
```

**ClassAssignment** (`app/Livewire/Admin/ClassAssignment.php`)
```php
Properties:
- $selectedStudent
- $selectedClass
- $conflicts (array)

Methods:
- checkConflicts()
- assignStudent()
- removeStudent($studentId, $classId)
```

**BulkAssignmentManager** (`app/Livewire/Admin/BulkAssignmentManager.php`)
```php
Properties:
- $selectedStudents (array)
- $targetClass
- $conflictReport

Methods:
- analyzeConflicts()
- processBulkAssignment()
```

### Teacher Components

**AttendanceMarker** (`app/Livewire/Teacher/AttendanceMarker.php`)
```php
Properties:
- $classId
- $scheduleId
- $students (collection)
- $attendanceData (array)

Methods:
- markAttendance($studentId, $status)
- bulkMarkPresent()
- saveAll()
```

**FaceRecognitionAttendance** (`app/Livewire/Teacher/FaceRecognitionAttendance.php`)
```php
Properties:
- $classId
- $sessionActive (boolean)
- $detectedStudents (array)
- $gracePeriodMinutes = 15

Methods:
- startSession()
- stopSession()
- processDetection($studentData) - called from JS
- manualOverride($studentId, $status)
```

### Shared Components

**NotificationCenter** (`app/Livewire/NotificationCenter.php`)
```php
Properties:
- $notifications
- $unreadCount

Methods:
- markAsRead($id)
- markAllAsRead()
- loadMore()

Events:
- Listens to: notification-received
```

**ScheduleCalendar** (`app/Livewire/ScheduleCalendar.php`)
```php
Properties:
- $userId
- $userType
- $currentWeek
- $schedules

Methods:
- render() - displays weekly schedule
- nextWeek(), previousWeek()
```

---

## 🔔 Events & Listeners

### Events (`app/Events/`)

**StudentEnrolled**
```php
Properties:
- public Student $student
- public Class $class
- public User $enrolledBy
```

**StudentRemoved**
```php
Properties:
- public Student $student
- public Class $class
```

**AttendanceMarked**
```php
Properties:
- public Attendance $attendance
- public bool $wasLate
```

**ClassScheduleChanged**
```php
Properties:
- public Class $class
- public array $oldSchedule
- public array $newSchedule
```

**TeacherAssignmentRequested**
```php
Properties:
- public Teacher $teacher
- public Student $student
- public Class $class
```

### Listeners (`app/Listeners/`)

**SendEnrollmentNotification**
```php
- Listens to: StudentEnrolled
- Notifies: Student (enrolled), Teacher (new student), Admin
- Via: DatabaseNotification, BrowserNotification
```

**SendRemovalNotification**
```php
- Listens to: StudentRemoved
- Notifies: Student, Teacher, Admin
```

**SendAttendanceNotification**
```php
- Listens to: AttendanceMarked
- Notifies: Student (if late/absent)
- Logic: Only send if status is late or absent
```

**SendScheduleChangeNotification**
```php
- Listens to: ClassScheduleChanged
- Notifies: All enrolled students, Teacher
```

**NotifyAdminOfAssignmentRequest**
```php
- Listens to: TeacherAssignmentRequested
- Notifies: Admin(s)
```

---

## 📢 Notifications (`app/Notifications/`)

**StudentEnrolledNotification**
```php
via(): ['database', 'broadcast']
toBroadcast(): Returns BroadcastMessage
toArray(): {
    'type' => 'enrollment',
    'title' => 'New Enrollment',
    'message' => 'You have been enrolled in {class_name}',
    'class_id' => ...,
    'url' => route('student.classes.show', ...)
}
```

**AttendanceMarkedNotification**
```php
via(): ['database', 'broadcast']
toArray(): {
    'type' => 'attendance',
    'status' => 'late|absent',
    'message' => 'You were marked {status} for {class_name}',
    'date' => ...,
}
```

**ScheduleChangedNotification**
```php
via(): ['database', 'broadcast']
toArray(): {
    'type' => 'schedule_change',
    'class_name' => ...,
    'changes' => [...],
}
```

---

## 🛣️ Routes (`routes/web.php`)

```php
// Public Routes
Route::get('/', function () { return redirect('/login'); });

// Auth Routes (customized)
Route::middleware('guest')->group(function () {
    Route::get('/login', [AuthController::class, 'showLogin'])->name('login');
    Route::post('/login', [AuthController::class, 'login']);
});

Route::post('/logout', [AuthController::class, 'logout'])->name('logout');

// Admin Routes
Route::middleware(['auth', 'role:admin'])->prefix('admin')->name('admin.')->group(function () {
    Route::get('/dashboard', [AdminDashboardController::class, 'index'])->name('dashboard');
    
    Route::resource('students', StudentController::class);
    Route::post('students/bulk-import', [StudentController::class, 'processBulkImport'])->name('students.bulk-import');
    
    Route::resource('classes', ClassController::class);
    Route::get('classes/{id}/roster', [ClassController::class, 'roster'])->name('classes.roster');
    
    Route::prefix('assignments')->name('assignments.')->group(function () {
        Route::get('/', [ClassAssignmentController::class, 'index'])->name('index');
        Route::post('/assign', [ClassAssignmentController::class, 'assign'])->name('assign');
        Route::post('/bulk-assign', [ClassAssignmentController::class, 'bulkAssign'])->name('bulk-assign');
        Route::delete('/{student}/{class}', [ClassAssignmentController::class, 'remove'])->name('remove');
    });
    
    Route::resource('rooms', RoomController::class)->except(['show']);
});

// Teacher Routes
Route::middleware(['auth', 'role:teacher'])->prefix('teacher')->name('teacher.')->group(function () {
    Route::get('/dashboard', [TeacherDashboardController::class, 'index'])->name('dashboard');
    
    Route::get('/classes', [TeacherClassController::class, 'index'])->name('classes.index');
    Route::get('/classes/{id}', [TeacherClassController::class, 'show'])->name('classes.show');
    
    Route::prefix('attendance')->name('attendance.')->group(function () {
        Route::get('/class/{classId}', [AttendanceController::class, 'index'])->name('index');
        Route::get('/class/{classId}/schedule/{scheduleId}/mark', [AttendanceController::class, 'create'])->name('create');
        Route::post('/class/{classId}/schedule/{scheduleId}', [AttendanceController::class, 'store'])->name('store');
        Route::put('/{id}', [AttendanceController::class, 'update'])->name('update');
        
        // Face Recognition
        Route::post('/class/{classId}/schedule/{scheduleId}/face-start', [AttendanceController::class, 'startFaceRecognition'])->name('face.start');
        Route::post('/face-record', [AttendanceController::class, 'recordFaceAttendance'])->name('face.record');
    });
    
    Route::post('/assignment-requests', [AssignmentRequestController::class, 'store'])->name('assignment-requests.store');
});

// Student Routes
Route::middleware(['auth', 'role:student'])->prefix('student')->name('student.')->group(function () {
    Route::get('/dashboard', [StudentDashboardController::class, 'index'])->name('dashboard');
    
    Route::get('/classes', [StudentClassController::class, 'index'])->name('classes.index');
    Route::get('/classes/{id}', [StudentClassController::class, 'show'])->name('classes.show');
    
    Route::get('/attendance', [StudentAttendanceController::class, 'index'])->name('attendance.index');
});

// Shared Routes
Route::middleware('auth')->group(function () {
    Route::post('/notifications/{id}/read', [NotificationController::class, 'markAsRead'])->name('notifications.read');
    Route::post('/notifications/read-all', [NotificationController::class, 'markAllAsRead'])->name('notifications.read-all');
});
```

---

## 🚀 Implementation Roadmap (1-Day Build)

### Phase 1: Foundation (2 hours)
1. ✅ Fresh Laravel install
2. ✅ Install Livewire, Tailwind CSS
3. ✅ Create all migrations
4. ✅ Run migrations, seed initial admin user
5. ✅ Setup authentication with role-based middleware
6. ✅ Create all models with relationships

### Phase 2: Admin Panel (3 hours)
7. ✅ Student CRUD (Controller + Views + Form Requests)
8. ✅ Class CRUD with schedule builder
9. ✅ Room management
10. ✅ Class Assignment Livewire component
11. ✅ Admin dashboard with stats

### Phase 3: Teacher Features (2 hours)
12. ✅ Teacher dashboard
13. ✅ View assigned classes
14. ✅ Manual attendance marking (Livewire component)
15. ✅ Basic attendance reports

### Phase 4: Student Features (1 hour)
16. ✅ Student dashboard
17. ✅ View enrolled classes with schedule calendar
18. ✅ View own attendance records

### Phase 5: Notifications (1.5 hours)
19. ✅ Setup events & listeners
20. ✅ Create notification classes
21. ✅ Notification dropdown Livewire component
22. ✅ Browser notifications (Pusher/Laravel Echo or polling)

### Phase 6: Face Recognition (2 hours - UNIQUE FEATURE)
23. ✅ Face recognition UI (camera access via WebRTC)
24. ✅ Attendance recording API endpoint
25. ✅ Auto-late marking logic (15-min grace period)
26. ✅ Teacher manual override functionality

### Phase 7: Polish (1.5 hours)
27. ✅ Conflict detection algorithm for bulk assignment
28. ✅ PDF export for class roster
29. ✅ Simple analytics charts on dashboards
30. ✅ Mobile-responsive adjustments
31. ✅ Testing & bug fixes

**Total: ~13 hours** (comfortable 1-day challenge with breaks)

---

## 🎨 Key Algorithms

### Schedule Conflict Detection
```php
function hasScheduleConflict($studentId, $newClassId) {
    $newSchedules = Schedule::where('class_id', $newClassId)->get();
    
    $existingClasses = User::find($studentId)
        ->classes()
        ->with('schedules')
        ->get();
    
    foreach ($existingClasses as $existingClass) {
        foreach ($existingClass->schedules as $existing) {
            foreach ($newSchedules as $new) {
                if ($existing->day_of_week === $new->day_of_week) {
                    // Check time overlap
                    if ($existing->start_time < $new->end_time && 
                        $existing->end_time > $new->start_time) {
                        return true; // Conflict detected
                    }
                }
            }
        }
    }
    
    return false;
}
```

### Auto-Late Marking
```php
function determinateAttendanceStatus($checkInTime, $scheduleStartTime) {
    $gracePeriod = 15; // minutes
    $diff = $checkInTime->diffInMinutes($scheduleStartTime);
    
    if ($diff <= $gracePeriod) {
        return 'present';
    } else {
        return 'late';
    }
}
```

### Bulk Assignment with Conflict Report
```php
function bulkAssignWithReport($studentIds, $classId) {
    $results = [
        'successful' => [],
        'conflicts' => [],
        'capacity_exceeded' => false,
    ];
    
    $class = Class::find($classId);
    $currentEnrolled = $class->students()->count();
    
    if (($currentEnrolled + count($studentIds)) > $class->capacity) {
        $results['capacity_exceeded'] = true;
        return $results;
    }
    
    foreach ($studentIds as $studentId) {
        if ($this->hasScheduleConflict($studentId, $classId)) {
            $results['conflicts'][] = [
                'student' => User::find($studentId),
                'conflict_classes' => $this->getConflictingClasses($studentId, $classId),
            ];
        } else {
            $class->students()->attach($studentId, [
                'enrolled_at' => now(),
                'assigned_by' => auth()->id(),
            ]);
            
            event(new StudentEnrolled(User::find($studentId), $class, auth()->user()));
            
            $results['successful'][] = User::find($studentId);
        }
    }
    
    return $results;
}
```

---

## 📦 Required Packages

```json
{
    "require": {
        "laravel/framework": "^10.0",
        "livewire/livewire": "^3.0",
        "barryvdh/laravel-dompdf": "^2.0",
        "spatie/laravel-permission": "^5.0" (optional, for advanced roles)
    },
    "require-dev": {
        "laravel/breeze": "^1.0"
    }
}
```

**NPM:**
```json
{
    "devDependencies": {
        "@tailwindcss/forms": "^0.5",
        "alpinejs": "^3.0",
        "autoprefixer": "^10.0",
        "postcss": "^8.0",
        "tailwindcss": "^3.0"
    }
}
```

---

## 🔐 Middleware

**RoleMiddleware** (`app/Http/Middleware/RoleMiddleware.php`)
```php
public function handle($request, Closure $next, $role) {
    if (!auth()->check() || auth()->user()->role !== $role) {
        abort(403, 'Unauthorized');
    }
    return $next($request);
}
```

Register in `app/Http/Kernel.php`:
```php
protected $middlewareAliases = [
    'role' => \App\Http\Middleware\RoleMiddleware::class,
];
```

---

## 📁 Additional Files to Create

### Seeders
- `DatabaseSeeder.php` - Run all seeders
- `AdminUserSeeder.php` - Create default admin
- `RoomSeeder.php` - Sample rooms
- `DemoDataSeeder.php` - Sample students, classes (for testing)

### Factories
- `UserFactory.php` (extend for student/teacher)
- `ClassFactory.php`
- `ScheduleFactory.php`
- `AttendanceFactory.php`

### Helpers
- `app/Helpers/StudentIdGenerator.php` - Generate E23-00345 format
- `app/Helpers/ScheduleHelper.php` - Schedule utilities

### Config
- Update `config/filesystems.php` for photo uploads
- Create `config/student.php` for student ID format, grace period settings

---

## 🎯 Success Criteria

✅ **Core Features:**
- Admin can manage students, classes, rooms
- Conflict-free class assignment with validation
- Teacher can mark attendance manually
- Students can view their schedule and attendance

✅ **Unique Feature:**
- Face recognition attendance via mobile camera
- Auto-detection of late arrivals
- Teacher override capability

✅ **Notifications:**
- Real-time in-app notifications
- Browser notifications
- Events triggering on enrollment, attendance, schedule changes

✅ **Quality:**
- Clean, maintainable code structure
- Responsive Tailwind UI
- Proper validation and error handling
- Role-based access control

---

## 📚 Next Steps

1. **Review this architecture** - Any changes needed?
2. **Setup environment** - Laravel install, Livewire, Tailwind
3. **Start with migrations** - Database first
4. **Follow the roadmap** - Phase by phase
5. **Test incrementally** - After each phase

---

**Ready to start building? Let me know if you need:**
- Migration files
- Model code
- Specific controller implementations
- Livewire component code
- Blade templates
- Or start from Phase 1! 🚀