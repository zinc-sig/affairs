---
authors: Kristopher Lam
state: prediscussion
discussion:
labels: process, ux
---

# Frontend Quality Assurance Statement (QAS)

### Stakeholders
Here we describe the roles that users of this system is likely to be apart in but not exclusive to
- Student
- Instructor
- Teaching Associates

### Functionalities guarantees

- Means to submission of work
> This describes the ways to transfer the source code files from users' end to the system. The web application is the current primary interface for uploading work to be recognized as a submission. However it is favorable to introduce another tier of interface such as CLI as a fallback in case of availability issue. 

- Viewing of graded reports and execution logs replay
> This describes the production of both machine-readable and the henceforth transformation into human readable form of grading result detailed with scoring of miscellaneous test cases and static / dynamic analytical feedback. Grading execution logs replay enhances the transparency of the grading process.

- System Status Monitoring
> This describes the ability for user to monitor the status of the system. This includes the ability to view the current queue of grading process, and the current status of the overall system health. This is to facilitate and reduce the process of outage reporting and incident response.
 
- Retreival of batch submission archives
> This describes the ability to retrieve a batch or a single submission to cross check the integrity of the transmission and storage facility with users' filesystem.

- Basic grading spreadsheet export
> This describes the ability to export a basic spreadsheet of grading result for the purpose of academic reporting logistics.
