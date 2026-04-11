# Fan Inspection System: Multi-Station Integration Proposal

## 1. Executive Summary

As our production requirements expand, we are proposing an upgrade to our current Fan Packaging Inspection System. We currently use a single station (`Ct-PlantProject`) that performs a final check on accessories and captures an image of the QR code. 

To improve traceability and coverage, the client requires a new **Initial Station** placed at the beginning of the belt. This station will actively scan the QR code to extract rich information (rather than just taking a picture of it) and verify the placement of the Fan Motor. This document outlines the proposed architecture to seamlessly integrate this new station with our existing Edge-to-Cloud infrastructure.

---

## 2. Current System Architecture

Currently, the system operates as a single edge node capturing all information at the end of the line and syncing it linearly to the cloud.

### How it Works Now
1. The fan box travels down the conveyor belt.
2. At the inspection station (`Ct-PlantProject`), the camera detects the presence of accessories (Shackle, Downrod, Canopy).
3. A secondary camera takes a static image of the QR label (but does not decode it).
4. The station bundles the inspection image, the QR image, and a pass/fail JSON result.
5. This bundle is pushed to AWS S3, where the Next.js Dashboard (`ct-inspection-logs`) visualizes it.

### Current Workflow Diagram
```mermaid
flowchart LR
    Box[Fan Box on Belt] --> StationA

    subgraph edge_system [Ct-PlantProject]
    StationA[Camera Station]
    Detect[AI: Detects Parts]
    QR[Camera: Captures QR Image]
    StationA --> Detect
    StationA --> QR
    end

    Detect --> Sync[AWS Sync Service]
    QR --> Sync

    Sync -- "Uploads Images + JSON" --> S3[(AWS S3)]
    S3 --> Dash[Dashboard]
```

---

## 3. Proposed Multi-Station Architecture

We will separate the responsibilities into two distinct edge stations. They will operate independently on the factory floor but their data will be unified in the AWS Cloud to present a single, complete history for every scanned box.

### How it Will Work
1. **Station 1 (New - Motor & QR Station)**: Placed at the beginning of the belt. A dedicated QR Scanner instantly extracts the actual data from the code (e.g., batch ID, serial number). The camera detects the Fan Motor. This station uploads a `motor_inspection.json` to a unique folder in AWS S3 based on the QR serial number.
2. **Station 2 (Existing - Accessories Station)**: Placed further down the belt. We will remove the QR camera from this station. It will identify the box passing through (either by tracking its position or a simple secondary scan) and verify the final accessories. It will upload `accessories_inspection.json` to the *same* folder in AWS S3.
3. **AWS S3 Integration**: S3 acts as the unifying brain. Because both stations use the QR code serial number as their ID, the files merge seamlessly.
4. **Cloud Dashboard**: The `ct-inspection-logs` dashboard will be updated to display both the Motor check and the Accessories check side-by-side for a completely traceable lifecycle.

### Proposed Workflow Diagram
```mermaid
flowchart TD
    Box1[Box Enters Line] --> QR
    
    subgraph station_1 ["Station 1: Motor and QR New"]
        QR[Hardware QR Scanner] --> Ex[Extracts Serial ID]
        Cam1[Motor Camera AI] --> DetectMotor[Detects Fan Motor]
        Ex --> Sync1[Upload motor_inspection.json]
        DetectMotor --> Sync1
    end

    Sync1 -- "S3 Key: /ID/motor.json" --> S3[(AWS S3 Unified Bucket)]

    subgraph station_2 ["Station 2: Accessories Existing"]
        Box2[Box Reaches Exit] --> Cam2
        Cam2[Accessories Camera AI] --> DetectAcc[Detects Parts]
        DetectAcc --> Sync2[Upload accessories_inspection.json]
    end

    station_1 -.->|"Conveyor Belt"| station_2
    Sync2 -- "S3 Key: /ID/accessories.json" --> S3

    S3 --> Dashboard[Inspection Dashboard]
    
    style S3 fill:#f96,stroke:#333,stroke-width:2px
```

---

## 4. The Offline Sync Challenge

**The Problem:** 
Factory internet connections can be intermittent, particularly when relying on Wi-Fi that requires workers to perform daily authenticated logins. When the cloud is unreachable, edge stations must store their inspection logs in a local database before eventually syncing to AWS. 

If Station 1 and Station 2 operate entirely offline and we remove the QR scanner from Station 2 (as originally proposed), a critical issue arises: **How does Station 2 know the QR ID of the physical box it is inspecting?** Without the ID, Station 2 cannot save its local data correctly, and AWS S3 will not be able to merge the Accessories check with the Motor check when the internet is restored.

To solve this, we must select one of the following three architectural approaches for local synchronization:

### Approach 1: The "Dual-Scanner" Strategy (Recommended)
Both stations are equipped with their own hardware QR scanners and operate 100% independently off their own local databases. They do not communicate with each other over the local network. 

* **Pros:** Most robust. Immune to local network failures and human intervention (e.g., if a box is removed from the belt between stations).
* **Cons:** Requires the purchase and maintenance of a second QR scanner.

```mermaid
flowchart LR
    subgraph offline ["Offline Factory Environment"]
        direction LR
        Box[Fan Box]
        
        subgraph s1 ["Station 1 (Motor)"]
            QR1[QR Scanner] --> DB1[(Local DB)]
            Cam1[Motor AI] --> DB1
        end
        
        subgraph s2 ["Station 2 (Accessories)"]
            QR2[QR Scanner] --> DB2[(Local DB)]
            Cam2[Accessory AI] --> DB2
        end
        
        Box --> QR1
        Box -. "Belt" .-> QR2
    end
    
    DB1 -- "Internet\nRestored" --> S3[(AWS S3\nUnified Bucket)]
    DB2 -- "Internet\nRestored" --> S3
```

### Approach 2: The "LAN Queue" Strategy
We retain a single QR scanner at Station 1. When Station 1 scans a box, it sends a payload over the Local Area Network (LAN) to a queue system living on Station 2. 

* **Pros:** Only requires one QR scanner.
* **Cons:** If boxes are removed or fall off the belt between the two stations, Station 2 will assign the wrong ID to the wrong box, completely breaking analytics.

```mermaid
flowchart TD
    subgraph LAN ["Offline Local Network (No AWS)"]
        Box[Fan Box]
        
        subgraph s1 ["Station 1 (Motor)"]
            QR1[QR Scanner: ID 123] --> DB1[(Local DB)]
            Cam1[Motor AI] --> DB1
        end
        
        s1 -- "LAN Event: Next is 123" --> Q[Station 2 Queue]
        
        subgraph s2 ["Station 2 (Accessories)"]
            Q --> NextBox[Assign ID 123]
            Cam2[Accessory AI] --> NextBox
            NextBox --> DB2[(Local DB)]
        end
        
        Box --> QR1
        Box -. "Travels on Belt" .-> Cam2
    end
    
    DB1 -- "Internet Restored" --> S3[(AWS S3 Unified Bucket)]
    DB2 -- "Internet Restored" --> S3
```

### Approach 3: The "Shared Local Database" Strategy
A single, centralized local database (e.g., hosted on Station 2's PC or a dedicated factory server) is used by both stations to sync data directly via the LAN. 

* **Pros:** Single source of truth locally. One background service handles all AWS uploads.
* **Cons:** Still requires a tracking mechanism (like the queue in Approach 2) if Station 2 doesn't have a scanner, and is vulnerable to LAN network crashes.

```mermaid
flowchart TD
    subgraph LAN ["Offline Local Network"]
        Box[Fan Box]
        
        subgraph s1 ["Station 1 (Motor)"]
            QR1[QR Scanner] --> D1[Motor Payload]
            Cam1[Motor AI] --> D1
        end
        
        subgraph s2 ["Station 2 (Master / Accessories)"]
            Cam2[Accessory AI] --> D2[Accessory Payload]
            MasterDB[(Shared Local DB)]
        end
        
        D1 -- "LAN Connection" --> MasterDB
        D2 --> MasterDB
        Box --> QR1
        Box -. "Belt" .-> Cam2
    end
    
    MasterDB -- "Internet Restored" --> S3[(AWS S3 Unified Bucket)]
```

---

## 5. Key Implementation Steps

1. **New Edge App Development (Station 1):**
   - Create a lightweight desktop app for the initial station.
   - Integrate hardware QR scanner support (via USB/Serial) to capture exact text strings rather than images.
   - Train and implement a YOLO model specifically for Fan Motor detection.
   - Configure local database fallback and S3 upload logic to use the extracted QR string as the primary `inspection_id`.

2. **Modify Existing App (`Ct-PlantProject`):**
   - Update hardware (either add scanner or integrate LAN network logic based on chosen approach).
   - Update the S3 synchronization to append its results directly into the folder created/identified by Station 1.

3. **Dashboard Enhancements (`ct-inspection-logs`):**
   - Modify the dashboard to pull *both* JSON files per inspection ID.
   - Design a timeline UI showing a box successfully passing Station 1 and subsequently Station 2.

## 6. Strategic Benefits
- **Zero Database Bottlenecks**: By continuing to use AWS S3 as the integration layer, we avoid costly RDS or DynamoDB setups.
- **Data Accuracy**: Hard-scanning the QR code is more reliable for analytics than taking photos of it.
- **Resilience**: Leveraging local databases ensures smooth factory operations despite captive-portal Wi-Fi issues.
- **Traceability**: If a box fails, the dashboard will immediately show exactly *which* station it failed at, pinpointing assembly line bottlenecks.
