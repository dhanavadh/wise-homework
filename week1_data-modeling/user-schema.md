# Data Modeling – Mentee
This section will be seperated into 4 units. Register, Dashboard, Req Mentoring Session, Mentoring Session

## 1. Mentee Register
From firestore pricing consideration, we may embed the mentee data into 1 i/o
```mermaid
erDiagram
    USER {
        string _id PK
        string email UK "unique index"
        string password_hash
        string role "mentor, mentee, admin, superadmin"
        string phone_number
        string profile_image
        datetime created_at
        datetime updated_at
    }

    LINE {
        string user_id UK "LINE user ID"
        string user_name
    }

    NAME {
        string prefix "Mr., Ms."
        string fullname_th "user input"
        string first_th "extracted from fullname_th ?"
        string last_th "extracted from fullname_th ?"
        string fullname_en "user input"
        string first_en "extracted from fullname_en ?"
        string last_en "extracted from fullname_en ?"
        string nickname_th
        string nickname_en
    }

    ACADEMIC_INFO {
        string faculty
        string section
        string referee
        string id_card_url
        string resume_url
    }

    ATTACHMENT {
        string label "transcript, certificate, etc."
        string url
    }

    MENTEE_STATUS {
        string type "student OR alumni"
        string student_id "only if type = student"
        number year "only if type = student"
        string id_number "only if type = alumni"
        number grad_year "only if type = alumni"
    }

    USER ||--|| LINE : "embeds"
    USER ||--|| NAME : "embeds"
    USER ||--o| ACADEMIC_INFO : "embeds (mentee only)"
    USER ||--o| MENTEE_STATUS : "embeds (mentee only)"
    ACADEMIC_INFO ||--o{ ATTACHMENT : "embeds"
```

```json
{
    // Authentication
    "email": "string",
    "password_hash": "string",
    "role": "string",

    // LINE Integration
    "line": {
        "user_id": "string",
        "user_name": "string",
    },

    // Personal Information
    "name": {
        "prefix": "string",
        "first_th": "string",
        "last_th": "string",
        "first_en": "string",
        "last_en": "string",
        "nickname_th": "string",
        "nickname_en": "string"
    },

    "phone_number": "string",
    "profile_image": "string",

    // Academic Information
    "academic_info": {
        "faculty": "string",
        "section": "string",
        "referee": "string",
        "id_card_url": "string",
        "resume_url": "string",
        "attachments": [
            {
                "label": "string",
                "url": "string"
            }
        ]
    },

    // Mentee Status
    "mentee_status": {
        "type": "string",

        // Student-only
        "student_id": "string",
        "year": "number",

        // Alumni-only
        "id_number": "string",
        "grad_year": "number"
    },

    "created_at": "datetime",
    "updated_at": "datetime"
}

```


### Key Consideration – Feb 11, 2026
1. Line Payload [See More @line-dev](https://developers.line.biz/en/docs/basics/user-profile/#profile-information-types). I haven't added any LINE user information into this data types.
2. Embedding or Referencing

---

## 2. Session


```json
{
    // Session ID
    "_id": "ObjectId",
    "mentee_id": "ObjectId",
    // Basic Info
    "topic": "string",
    "description": "string",
    "expectation": "string",
    "preferred_time": "string",
    
    // Mentor Selection
    "mentor_selections": [
        {
            "rank": 1,
            "mentor_id": "ObjectId",
            "status": "string"
        }
    ],

    "status": "string",
    // Matched Mentor
    "matched_mentor_id": "ObjectId",

    // Mentee Quota
    "user_quota_id": "ObjectId",
    // Appointments
    "appointments": [
        {
            "appointment_id": "ObjectId",
            "appointment_number": 1,
            "scheduled_date": "datetime",
            "previous_scheduled_date": "datetime"
            "duration_minutes": "number",
            "meeting_url": "string",
            "status": "string",
            "cancelled_by": "string",
            "notes": "string",
            "mentee_feedback": {
                "rating": "number",
                "comment": "string"
            },
            "mentor_feedback": {
                "comment": "string",
                "follow_up_needed": "boolean"
            },
            "created_at": "datetime",
            "updated_at": "datetime"
        }
    ],

    // Metadata
    "created_at": "datetime",
    "updated_at": "datetime",
    "submitted_at": "datetime",
    "matched_at": "datetime",
    "completed_at": "datetime"
}

```


```mermaid
erDiagram
    SESSION {
        string _id PK
        string mentee_id FK "ref to users"
        string topic "5 predefined topics"
        string description "max 300 words"
        string expectation "max 300 words"
        string preferred_time "free text availability"
        string status "draft, submitted, matched, in_progress, completed, cancelled, expired"
        string matched_mentor_id FK "ref to users"
        string user_quota_id FK "ref to quotas"
        datetime created_at
        datetime updated_at
        datetime submitted_at
        datetime matched_at
        datetime completed_at
    }

    MENTOR_SELECTION {
        number rank "1-5 priority order"
        string mentor_id FK "ref to users"
        string status "pending, accepted, declined"
    }

    APPOINTMENT {
        string appointment_id PK
        number appointment_number "1st, 2nd, 3rd meeting"
        datetime scheduled_date
        datetime previous_scheduled_date "original date if rescheduled"
        number duration_minutes
        string meeting_url "gg meet link"
        string status "scheduling, confirmed, completed, cancelled, no_show, rescheduled"
        string cancelled_by "mentee, mentor, or system"
        string notes "post-meeting notes"
        datetime created_at
        datetime updated_at
    }

    MENTEE_FEEDBACK {
        number rating "1-5"
        string comment
    }

    MENTOR_FEEDBACK {
        string comment
        boolean follow_up_needed
    }

    SESSION ||--|{ MENTOR_SELECTION : "embeds"
    SESSION ||--o{ APPOINTMENT : "embeds"
    APPOINTMENT ||--o| MENTEE_FEEDBACK : "embeds"
    APPOINTMENT ||--o| MENTOR_FEEDBACK : "embeds"
```


### Consideration
- Quota system >> Need ref due to frequent update?
- feedback has 3 points (ref from Figma)