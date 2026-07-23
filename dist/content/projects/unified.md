# Unified API

Learning Management System (LMS) designed for educational institutions. It simplifies **course management**, **assessments**, and **academic workflows** by bringing core learning and administrative functions into a single, cohesive system.

## Contributors
- **Backend Contributor**: [@th0truth](https://github.com/th0truth)
- **Frontend Contributor**: [@Lendoker](https://github.com/Lendoker)

## Web Client Screenshots
*Click on any image to preview in full-screen overlay mode:*

<div class="screenshots-group">
  <div class="screenshot-item" data-src="https://raw.githubusercontent.com/UnifiedLMS/unified-web/main/.github/assets/unified-login.png">
    <img src="https://raw.githubusercontent.com/UnifiedLMS/unified-web/main/.github/assets/unified-login.png" alt="Authentication" />
    <figcaption>Authentication & Google Sign-In</figcaption>
  </div>
  <div class="screenshot-item" data-src="https://raw.githubusercontent.com/UnifiedLMS/unified-web/main/.github/assets/unified-student-home.png">
    <img src="https://raw.githubusercontent.com/UnifiedLMS/unified-web/main/.github/assets/unified-student-home.png" alt="Student Dashboard" />
    <figcaption>Student Workspace</figcaption>
  </div>
  <div class="screenshot-item" data-src="https://raw.githubusercontent.com/UnifiedLMS/unified-web/main/.github/assets/unified-student-schedule.png">
    <img src="https://raw.githubusercontent.com/UnifiedLMS/unified-web/main/.github/assets/unified-student-schedule.png" alt="Student Schedule" />
    <figcaption>Timetable Schedule</figcaption>
  </div>
  <div class="screenshot-item" data-src="https://raw.githubusercontent.com/UnifiedLMS/unified-web/main/.github/assets/unified-student-grades.png">
    <img src="https://raw.githubusercontent.com/UnifiedLMS/unified-web/main/.github/assets/unified-student-grades.png" alt="Student Grades" />
    <figcaption>Student Grades View</figcaption>
  </div>
  <div class="screenshot-item" data-src="https://raw.githubusercontent.com/UnifiedLMS/unified-web/main/.github/assets/unified-teacher-grades.png">
    <img src="https://raw.githubusercontent.com/UnifiedLMS/unified-web/main/.github/assets/unified-teacher-grades.png" alt="Teacher Workspace" />
    <figcaption>Teacher Gradebook</figcaption>
  </div>
  <div class="screenshot-item" data-src="https://raw.githubusercontent.com/UnifiedLMS/unified-web/main/.github/assets/unified-teacher-schedule-edit.png">
    <img src="https://raw.githubusercontent.com/UnifiedLMS/unified-web/main/.github/assets/unified-teacher-schedule-edit.png" alt="Schedule Edit" />
    <figcaption>Teacher Schedule Editor</figcaption>
  </div>
</div>

## Related Repositories
- **API Backend**: [UnifiedLMS/unified-api](https://github.com/UnifiedLMS/unified-api)
- **Web Frontend**: [UnifiedLMS/unified-web](https://github.com/UnifiedLMS/unified-web)

## Stack
- **FastAPI**
- **MongoDB**
- **Redis**
- **OAuth 2.0 / JWT / Google Auth**
- **Docker & Nginx**
- **Pytest & UV**

## Highlights
- **Async API Design**: High-throughput asynchronous endpoints built with FastAPI.
- **Role-Based Auth**: Secure multi-tier permissions for Students, Instructors, and Admins.
- **Rate Limiting & Caching**: Protected via Redis in-memory storage.
- **Containerized Deployment**: Production-ready Docker & Nginx orchestration.
