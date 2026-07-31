# Academic Hub Documentation Repository

This repository contains the product, architecture, and workflow documentation for the Academic Hub platform. It is a documentation-first workspace for requirements, technical design, use cases, policy amendments, and compatibility audits.

## What this repository contains

The documents in this repository define the agreed scope and implementation direction for the platform across three roles:
- Student
- Lecturer
- Admin

The content covers core product requirements, database design, scheduling, notifications, file management, room management, and business policies.

## Document map

### Product and requirements
- [MVP_Functions_V3.6_Professional.md](MVP_Functions_V3.6_Professional.md) — approved MVP scope, functional requirements, and product decisions.
- [Amendment_Subscription_Blocking_Policy_v2.0.md](Amendment_Subscription_Blocking_Policy_v2.0.md) — formal amendment to subscription blocking and access rules.

### Technical design documents
- [TDD_Core_Data_Layer_V2.1_Unified.md](TDD_Core_Data_Layer_V2.1_Unified.md) — unified core data layer, schema, RLS, multi-tenancy, and system-wide design decisions.
- [TDD_Scheduling_and_Events_System.md](TDD_Scheduling_and_Events_System.md) — scheduling engine, event model, and calendar-related workflows.
- [TDD_Room_Management_V1.0.md](TDD_Room_Management_V1.0.md) — room and lab allocation, booking requests, and exam scheduling workflows.
- [TDD_Notifications_System_v1.1.md](TDD_Notifications_System_v1.1.md) — notifications, announcements, and delivery architecture.
- [TDD_Chat_System_v1.0.md](TDD_Chat_System_v1.0.md) — chat channels, posts, comments, and support-thread design aligned with the MVP and multi-tenancy rules.
- [tdd_file_repository_v1.1.md](tdd_file_repository_v1.1.md) — file repository design, upload/download flows, access policies, and review workflows.

### Use case documents
- [Use_Cases_Student_v2.0.md](Use_Cases_Student_v2.0.md) — student-facing use cases and workflows.
- [Use_Cases_Lecturer_v1.0.md](Use_Cases_Lecturer_v1.0.md) — lecturer-facing use cases and operational workflows.
- [Use_Cases_Admin_v1.0.md](Use_Cases_Admin_v1.0.md) — admin-facing use cases, governance, and management actions.

### Audits and references
- [Compatibility_Audit_FileRepo_Notifications_v1.0.md](Compatibility_Audit_FileRepo_Notifications_v1.0.md) — compatibility audit and required corrections across the file repository and notifications systems.
- [tdd_template.md](tdd_template.md) — reusable template for future technical design documents.

## Repository purpose

This repo serves as the single source of documentation for aligning product, engineering, and business stakeholders around the Academic Hub platform. It is intended to support planning, implementation, and future review of the platform’s core systems.

## Notes

- The repository is documentation-focused and does not contain application source code.
- Most documents are written in Arabic with English section titles and technical terminology.
- The content reflects a multi-tenant, education-platform architecture with strong requirements around access control, institutional isolation, and role-based workflows.
