# User Story

**Title:** View Patient Medical History

**User Story:**
As a **nurse**, I want to **view a patient's medical history**, so that **I can understand the patient's past conditions, treatments, and allergies before providing care**.

---

# Description

Nurses should be able to access a patient's complete medical history from the Electronic Health Record (EHR) system. This includes previous diagnoses, surgeries, medications, allergies, lab results, and hospitalization records. Access must be restricted to authorized healthcare staff.

---

# Actors

* Nurse
* EHR System

---

# Preconditions

* Nurse must be logged into the hospital system.
* Nurse must have authorization to access patient records.
* Patient record must exist in the system.

---

# Trigger

The nurse opens a patient profile from the **patient dashboard or search results**.

---

# Basic Flow

1. Nurse searches for a patient using patient ID or name.
2. System displays patient profile.
3. Nurse selects **Medical History** section.
4. System retrieves patient's historical records.
5. System displays medical history including:

   * diagnoses
   * past procedures
   * medications
   * allergies
   * lab results
6. Nurse reviews the information.

---

# Alternative Flows

### A1 – Patient record not found

1. Nurse searches patient.
2. System returns **no results**.
3. System displays message: *"Patient record not found."*

### A2 – Unauthorized access

1. Nurse attempts to open patient record.
2. System verifies permissions.
3. System denies access and displays error message.

---

# Acceptance Criteria (Gherkin)

```gherkin
Feature: View patient medical history

Scenario: Nurse views medical history successfully
Given the nurse is logged into the system
And the patient record exists
When the nurse opens the patient's medical history
Then the system should display the patient's diagnoses, medications, allergies, and procedures

Scenario: Patient record not found
Given the nurse searches for a patient
When no matching patient exists
Then the system should display "Patient record not found"

Scenario: Unauthorized access
Given the nurse attempts to view a patient record
When the nurse does not have permission
Then the system should deny access
```

---

# Business Rules

* Only authorized medical staff can access patient records.
* Patient medical history must include:

  * diagnosis history
  * medication history
  * allergy information
  * surgical procedures
* Access to patient data must be **logged for audit compliance**.

---

# Non-Functional Requirements

* Medical history should load within **3 seconds**.
* Access logs must be stored for **audit compliance**.
* Data must comply with **HIPAA/privacy regulations**.



