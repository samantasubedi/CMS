# Feature Requirements Document (FRD)
## Generic Dynamic Content Management System (CMS)

**Version:** 1.0 | **Scope:** Functional behavior of all major features (implementation-independent)

---

## 1. Feature Index

| # | Feature | Primary Actors |
|---|---------|-----------------|
| 1 | Authentication | All Users |
| 2 | User Management | Super Admin, Admin |
| 3 | Role & Permission Management | Super Admin |
| 4 | Content Type Management | Admin |
| 5 | Dynamic Schema Management | Admin |
| 6 | Content CRUD Operations | Admin, Editor, Contributor |
| 7 | Dynamic Form Generation | Admin, Editor, Contributor |
| 8 | Search, Filtering & Pagination | All Users |
| 9 | Activity Logging | System, Admin |
| 10 | Version History | Admin, Editor |
| 11 | Dashboard & Statistics | Admin |
| 12 | Content Publishing Workflow | Editor, Reviewer, Admin |
| 13 | Notifications | All Users |

---
## 2. Feature Specifications

### 2.1 Authentication
* **Objective:** Secure login, logout, and session management.
* **Flow:** 1. User submits credentials.
  2. Credentials are validated.
  3. Session is issued.
  4. User is redirected to the dashboard.
  * Note: Logout invalidates the session. Password reset occurs via an emailed, single-use, time-limited link.
* **Business Rules:**
  * Lock account for 15 min after 5 failed attempts in 10 min.
  * Reset links expire in 30 min.
  * Disabled accounts can't log in.
* **Validation:**
  * Valid email format.
  * Password must be ≥8 characters with 1 uppercase + 1 number.

| Failure Case | Expected Behavior |
| :--- | :--- |
| Invalid credentials | Generic error (no field hints) |
| Locked/disabled account | Explanatory message |
| Expired reset link | Re-request prompt |

* **Acceptance Criteria:**
  * Lockout triggers correctly.
  * Reset links expire on schedule.
  * Logout is immediate.
* **Dependencies:**
  * User Management.

---

### 2.2 User Management
* **Objective:** Create, update, deactivate, and manage user accounts.
* **Flow:** 1. Admin creates/edits users (name, email, role).
  2. New users start in "Pending" status until activation via email.
  3. Admin deactivates/reactivates/deletes accounts as needed.
  4. Users self-edit their own profile (cannot edit role).
* **Business Rules:**
  * No self-deactivation.
  * Sole Super Admin can't be demoted/deactivated.
  * Deleting a user retains their content (reassigned to placeholder/reference).
* **Validation:**
  * Unique email required.
  * Role must reference a valid existing role.

| Failure Case | Expected Behavior |
| :--- | :--- |
| Duplicate email | Creation rejected |
| Self-deactivation attempt | Operation blocked |
| Last Super Admin removal | Operation blocked |

* **Acceptance Criteria:**
  * Deactivated users can't authenticate.
  * Duplicate emails are rejected at creation.
* **Dependencies:**
  * Authentication
  * Role & Permission Management

---

### 2.3 Role & Permission Management
* **Objective:** Define roles and their permitted actions.
* **Flow:** 1. Super Admin views/creates/edits roles with granular permissions (content, users, settings).
  2. Super Admin assigns defined roles to users.
* **Business Rules:**
  * Default roles (Super Admin, Admin, Editor, Contributor, Viewer) can't be deleted.
  * Roles with assigned users can't be deleted until those users are reassigned.
  * Permissions are additive (no explicit deny in v1).
* **Validation:**
  * Unique role name.
  * ≥1 permission required per custom role.

| Failure Case | Expected Behavior |
| :--- | :--- |
| Role is in use or is a default role | Deletion is blocked |

* **Acceptance Criteria:**
  * Permission changes apply immediately without requiring re-login.
  * In-use/default roles are protected from deletion.
* **Dependencies:**
  * User Management.

---

### 2.4 Content Type Management
* **Objective:** Let admins define new content categories without developer involvement.
* **Flow:** 1. Admin creates a Content Type (name, slug, description).
  2. Admin adds fields to the Content Type.
  3. Admin publishes the Content Type.
  4. Content Type becomes available for content entry.
  * Note: Admins can edit metadata or archive the Content Type (archiving preserves existing entries as read-only).
* **Business Rules:**
  * Slugs must be unique and URL-safe.
  * Types with existing entries can't be hard-deleted (must archive instead).
  * Renaming a Content Type doesn't change its established slug.
* **Validation:**
  * Name must be 2–50 characters.
  * Slug must be lowercase alphanumeric + hyphens, and completely unique.

| Failure Case | Expected Behavior |
| :--- | :--- |
| Duplicate slug | Creation/update rejected |
| Delete attempted when entries exist | Deletion blocked; system prompts to archive instead |

* **Acceptance Criteria:**
  * Unpublished types remain unusable for entry creation.
  * Slug uniqueness is strictly enforced.
* **Dependencies:**
  * Dynamic Schema Management.

---

### 2.5 Dynamic Schema Management
* **Objective:** Define/modify the fields belonging to a Content Type at runtime.
* **Flow:** 1. Admin adds fields (type, label, constraints, default, options).
  2. Admin reorders the fields.
  3. Admin edits/removes fields as needed.
  4. System validates the entire schema pre-publish.
* **Business Rules:**
  * Field keys must be unique and are immutable once set.
  * Changing a field's data type after entries exist isn't allowed (a new field must be created instead).
  * A Content Type needs ≥1 field to publish.
  * New optional fields backfill existing entries with null/default values.
  * Removed fields are soft-deleted with a grace period.
* **Validation:**
  * Label must be 1–50 characters.
  * Select fields require ≥2 options.
  * Numeric min must be less than max (min < max).

| Failure Case | Expected Behavior |
| :--- | :--- |
| Type-change attempt on populated field | Operation blocked |
| Publish attempted with zero fields | Publish blocked |

* **Acceptance Criteria:**
  * Field key immutability is strictly enforced.
  * Existing entries survive field removal via soft-delete.
* **Dependencies:**
  * Content Type Management.

---

### 2.6 Content CRUD Operations
* **Objective:** Create, read, update, delete content entries per Content Type.
* **Flow:** 1. User selects a Content Type.
  2. User clicks "New Entry".
  3. Dynamic form is rendered to the user.
  4. User saves the entry as a Draft or submits it for Publish.
  5. User views/edits/deletes existing entries per assigned permission.
  * Note: Deletes go to the Trash before permanent removal.
* **Business Rules:**
  * Only Admin/Editor can hard-delete.
  * Contributors can only move entries to the Trash.
  * Entries can't be published with incomplete required fields.
  * Creator and last-editor are always tracked.
* **Validation:**
  * All schema-required fields must be filled before a publish action.
  * Field-level rules (min/max, format, uniqueness) are enforced upon saving.

| Failure Case | Expected Behavior |
| :--- | :--- |
| Missing required field | Blocks publishing capability |
| Unauthorized delete attempt | Request rejected |
| Editing a Trashed entry | Blocked until the entry is restored |

* **Acceptance Criteria:**
  * Incomplete required fields block publishing.
  * Soft-deleted entries are fully recoverable within the defined retention window.
  * Role-based edit scope is strictly enforced.
* **Dependencies:**
  * Dynamic Schema Management
  * Roles
  * Publishing Workflow

---

### 2.7 Dynamic Form Generation
* **Objective:** Auto-render a data-entry form from a Content Type's schema.
* **Flow:** 1. System reads the schema.
  2. System renders one control per field in the defined order (text → input, boolean → toggle, select → dropdown, relation → picker, image → uploader).
  3. Inline and submit-time validation are executed.
* **Business Rules:**
  * Relation fields restrict selection strictly to the target type's entries.
  * Unknown field types degrade gracefully (renders a placeholder, no application crash).
  * The form regenerates automatically upon any schema change.
* **Validation:**
  * Client-side validation mirrors schema rules.
  * Server-side validation during save functions as the final authority.

| Failure Case | Expected Behavior |
| :--- | :--- |
| Deleted relation target | Form displays a "target unavailable" state; other fields remain unaffected |
| Corrupted field type | Renders as read-only accompanied by a warning icon |

* **Acceptance Criteria:**
  * Forms update automatically after schema changes occur.
  * All valid field types successfully render an appropriate UI control.
* **Dependencies:**
  * Dynamic Schema Management
  * Content CRUD

---

### 2.8 Search, Filtering & Pagination
* **Objective:** Efficiently locate entries/users in large datasets.
* **Flow:** 1. User searches by keyword and/or applies filters (field value, status, date, author).
  2. User applies sorting parameters.
  3. User pages through the resulting dataset.
* **Business Rules:**
  * Default page size is 20 (maximum is configurable, e.g., up to 100).
  * Filters combine natively using AND logic.
  * Results are always strictly scoped to the viewer's explicit permissions.
* **Validation:**
  * Page and size values must be positive integers within allowable bounds.
  * Invalid filter fields are ignored rather than throwing an error.

| Failure Case | Expected Behavior |
| :--- | :--- |
| No matching results found | Displays a clear empty state |
| Out-of-range page requested | Returns an empty set accompanied by valid pagination metadata |

* **Acceptance Criteria:**
  * Combined filters narrow down datasets correctly.
  * Pagination metadata remains completely accurate.
  * No cross-permission data leakage occurs.
* **Dependencies:**
  * Content CRUD
  * Roles

---

### 2.9 Activity Logging
* **Objective:** Maintain an auditable record of significant system actions.
* **Flow:** 1. Any loggable action (login, content/schema/role/user change) auto-generates a log entry containing the actor, action, target, and timestamp.
  2. Admin views and/or filters the generated log.
* **Business Rules:**
  * Logs are append-only and strictly immutable by anyone, including the Super Admin.
  * Logs are retained for ≥12 months before being moved to archival.
  * Logging processes never block the underlying application action (best-effort execution).
  * No sensitive data (such as passwords) is ever recorded in logs.
* **Validation:**
  * Each log entry explicitly requires an actor, action type, target, and valid timestamp.

| Failure Case | Expected Behavior |
| :--- | :--- |
| Logging service failure | Does not block the primary action; the log failure itself is flagged for engineering review |

* **Acceptance Criteria:**
  * Logs are completely immutable.
  * Every significant system change successfully produces a log entry.
  * Credentials never appear within logs.
* **Dependencies:**
  * All other features (acting as the log sources).

---

### 2.10 Version History
* **Objective:** Preserve prior versions of an entry for review/rollback.
* **Flow:** 1. Each save operation snapshots the previous state of the entry.
  2. User views the chronological version list (displaying timestamp and editor attribution).
  3. User previews or chooses to restore a specific version.
  4. Restoring creates a new current version (entire preceding history is preserved, not erased).
* **Business Rules:**
  * System retains a capped number of versions per entry (e.g., last 50, followed by auto-pruning).
  * Only users possessing explicit edit permissions can restore a historical version.
* **Validation:**
  * Restoring requires a valid, existing version ID.

| Failure Case | Expected Behavior |
| :--- | :--- |
| Restoring a pruned version | Returns a "no longer available" error |
| Unauthorized restore attempt | Operation is blocked |

* **Acceptance Criteria:**
  * Every save successfully creates a distinct version snapshot.
  * Restoring a version never deletes subsequent history.
  * The history list renders chronologically with accurate user attribution.
* **Dependencies:**
  * Content CRUD
  * Activity Logging

---

### 2.11 Dashboard & Statistics
* **Objective:** Give role-appropriate, at-a-glance system/content overview.
* **Flow:** 1. User lands on the Dashboard page immediately post-login.
  2. User views metric cards (content counts by type/status, pending reviews) and a recent activity feed.
  3. User clicks through summary cards to access pre-filtered lists.
* **Business Rules:**
  * Metrics are scoped precisely to the viewer's permissions (e.g., Contributors see only their personal counts).
  * Admins see system-wide aggregated metrics.

| Failure Case | Expected Behavior |
| :--- | :--- |
| No system content generated yet | Renders a zero-state dashboard with a creation prompt instead of an error |

* **Acceptance Criteria:**
  * Dashboard metrics respect role scopes.
  * Metric card clicks navigate smoothly to correctly filtered lists.
  * Displays a graceful zero-state.
* **Dependencies:**
  * Content CRUD
  * Activity Logging
  * Roles

---

### 2.12 Content Publishing Workflow
* **Objective:** Govern entry lifecycle: Draft → In Review → Published → Archived.
* **Flow:** 1. Contributor drafts an entry and submits it for review.
  2. Reviewer either approves (moves to Published) or rejects with a mandatory comment (reverts to Draft).
  3. Admin may publish/unpublish directly if permitted by system configurations.
  4. Published entries can be moved to Archived (restorable back to Draft status only).
* **Business Rules:**
  * Only users with explicit "publish" permission can execute a publish action.
  * The original Contributor is locked out of edits while an entry is "In Review" (Reviewer/Admin can still edit).
  * Any rejection action strictly requires a descriptive comment.
* **Validation:**
  * Rejection comment must be non-empty.
  * Publishing re-validates all schema-required fields, regardless of prior validation states during Draft saving.

| Failure Case | Expected Behavior |
| :--- | :--- |
| Empty rejection comment | Operation blocked |
| Edit attempt on an entry that is "In Review" | Operation blocked |
| Publish attempted when data has since become invalid | Publish blocked (e.g., if a relation target was removed in the interim) |

* **Acceptance Criteria:**
  * Mandatory review workflows cannot be bypassed when configured.
  * Rejections always carry a mandatory comment.
  * "In Review" entries are securely locked from author edits.
* **Dependencies:**
  * Content CRUD
  * Roles
  * Notifications

---

### 2.13 Notifications
* **Objective:** Inform users of relevant events without manual polling.
* **Flow:** 1. Triggering event occurs (e.g., a new review request is submitted).
  2. Notification is generated automatically for the relevant recipient(s).
  3. User sees an unread indicator, views their list, and marks notifications as read.
  4. Clicking a notification navigates the user directly to the related content entry.
* **Business Rules:**
  * No self-notification is generated for one's own actions.
  * Notification history is retained for a defined period (e.g., 90 days) before systematic cleanup.
* **Validation:**
  * Notification must resolve to a valid system recipient and a valid target entity.
  * Orphaned notifications (where the target was deleted) are auto-removed or marked unavailable.

| Failure Case | Expected Behavior |
| :--- | :--- |
| Target entity is deleted | Displays "content no longer available" instead of rendering a broken link |
| Recipient permission is revoked | Notification is automatically hidden from view |

* **Acceptance Criteria:**
  * Self-notifications are blocked entirely.
  * Unread count indicators remain fully accurate.
  * Graceful handling is guaranteed for deleted targets.
* **Dependencies:**
  * Publishing Workflow
  * Roles
  * Activity Logging
## 3. Cross-Feature Business Rules Summary

| Rule | Applies To |
|------|------------|
| Soft-delete before hard-delete, with grace/retention period | Content CRUD, Schema fields, Users |
| Role-based visibility scoping on all list/search views | Search & Filtering, Dashboard, Notifications |
| No self-notification / no self-account-deactivation | Notifications, User Management |
| Immutable, append-only logs | Activity Logging |
| Required-field validation re-checked at publish time, not just save time | Content CRUD, Publishing Workflow |

---

