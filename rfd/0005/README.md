---
authors: Kristopher Lam
state: prediscussion
discussion:
labels: process, ux
---

# Frontend Quality Assurance Statement (QAS)

## Background
This document lays down the quality assurance statement for the frontend of the system. Further revision and amendment will be introduced in later RFDs as the frontend evolves. It serves as a reference for functional requirements and user experience during development.

## Stakeholders
Here we describe the roles that users of this system is likely to be apart in but not exclusive to
- Student
- Teaching Associates
- System Administrator

## Functionalities guarantees

- Configure grading pipeline workflow
> This describes the ability to configure the grading pipeline workflow, including stages of task to run, the options and test cases to specify in each stage and its dependencies and the sequence of execution.

- Assign assessment
> This describes the ability to assign assessment to students, including the ability to specify the assessment details, such as the assessment name, the assessment type, various due date and deadline, and grading policy associated with the assessment.

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

- Role and permission management
> This describes the ability to manage user accounts' role, including the ability to authorize and revoke access to certain resouces through the use of a web interface.

